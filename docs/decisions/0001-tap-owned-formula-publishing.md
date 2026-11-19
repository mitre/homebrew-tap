# ADR-0001: Tap-owned formula publishing — verified pull requests, not direct pushes

**Date:** 2026-11-18
**Status:** accepted
**Deciders:** Aaron Lippold (tap maintainer)

## Context

Source projects publish their formulae into this tap from their own release
workflows. Today the only one, mitre/claude-statusline, renders
`Formula/claude-statusline.rb` from its published `checksums.txt` and pushes it
straight to `main` with the tap-publisher GitHub App token. That leaves five
problems:

- **Nothing verifies a formula before users get it.** Pushes to `main` get syntax
  checks only. Moreover, `tests.yml` has never started a job: job-level
  `options:` is invalid, so every run since 2026-10-04 failed with zero jobs.
- **The generated formula would fail `brew audit`.** Its `version "0.4.0"`
  duplicates the version Homebrew reads from the URL. Homebrew's
  `resource_auditor.rb` reports that as "redundant with version scanned from
  URL".
- **Linux arm64 is never installed in CI**, though the formula ships it.
- **Each new source project would copy the render and publish scripts.** The
  tap epic `homebrew-tap-g9x` plans four more tools, and hdf-cli makes five.
- **Nothing proves a file was built by its source project's release workflow.**
  A sha256 proves only that the file did not change after the checksum was
  taken.

The tools to onboard come in two shapes:

| Tool | Source | Published as | Tag | Provenance today (checked 2026-11-18) |
|---|---|---|---|---|
| claude-statusline | mitre/claude-statusline | release archives + `checksums.txt` | `v0.4.0` | none |
| hdf-cli | mitre/hdf-libs (monorepo) | release archives `hdf_<ver>_<os>_<arch>.tar.gz` + `checksums.txt`, among other assets; `-rc` tags publish pre-releases | `v3.8.0` | `actions/attest-build-provenance` with `subject-checksums`, which attests each archive listed in `checksums.txt`; verifies with this ADR's flags |
| saf-cli | mitre/saf | npm `@mitre/saf` | `1.8.0` (no `v`) | npm trusted-publishing provenance; verifies with this ADR's flags |
| cis-bench | mitre/cis-bench | PyPI `cis-bench` | `v0.5.2`, pushed by semantic-release with `GITHUB_TOKEN` | PEP 740 attestation bound to `refs/heads/main`, not the tag |
| emasser | mitre/emasser | RubyGems `emasser` | `3.22.0` (no `v`) | none; published with an API key |
| heimdall-cli | mitre/heimdall-cli | nothing yet (Linux-only goreleaser config) | none | none |

Two facts from Homebrew's source (Homebrew/brew 179af39, read and confirmed
2026-11-18) shape the verification design:

- `brew test-bot --only-formulae --build-from-source` returns before fetch,
  install and test (`test_bot/formulae.rb` 712-725).
- Without that flag, test-bot ignores install, linkage and test failures for an
  existing formula with no bottle for its current version (lines 710 and
  801-860).

Our formulae are never bottled. So test-bot would report green on a broken bump.

## Decision

The tap owns publishing. A source release only signals the tap. The tap fetches
and verifies the release, updates the formula, and merges it through a pull
request that nothing can skip.

Each formula has a config file in the tap. Every config names `kind`
(`archive`, `npm`, `pypi` or `rubygems`), `source_repo`, `signer_workflow` and
`tag_template` (`v{version}` or `{version}`). The kind selects the fetcher, the
provenance input and how the formula is updated.

1. **Trigger.** After a stable source release is published, the source sends
   `repository_dispatch` (event type `formula-release`, payload
   `{formula, version}`) to this tap. Pre-release tags send nothing.
   - For `archive` kinds it is sent once the GitHub Release exists and its
     attestations are written.
   - For registry kinds it is sent from the job that published to the registry,
     after the publish succeeds. A GitHub Release is not proof the registry has
     the version: saf 1.8.0's release was published on 2026-11-02 and reached
     npm on 2026-11-17.
   - The same tap workflow accepts `workflow_dispatch` with those inputs, for
     re-runs.
2. **Availability.** The tap does not trust the dispatch. For registry kinds it
   polls the registry for the exact version, with a bounded wait, and fails the
   run if the version does not appear. It downloads the file the formula will
   use (npm tarball, PyPI sdist, `.gem`) and compares its digest with the
   registry's metadata (npm `dist.integrity`, PyPI `digests.sha256`, RubyGems
   `sha`). For `archive` kinds it compares each archive with `checksums.txt`.
3. **Provenance.** The tap resolves the tag (`tag_template` applied to the
   version) to its commit through the GitHub API. Then it runs
   `gh attestation verify` on every file the formula downloads, with all of
   these flags:
   - `--repo <source repo>`;
   - `--signer-workflow <source repo>/<signer workflow path>`;
   - `--source-ref refs/tags/<tag>`;
   - `--source-digest <tag commit>`;
   - `--deny-self-hosted-runners`.

   The ref and digest bind each file to that tag's build. Repository and
   workflow identity alone would also accept a file from any other run of the
   same workflow. A missing or failing attestation stops the bump; no PR is
   opened. Where the attestation comes from depends on the kind:
   - **archive:** the source release attests its archives with `actions/attest`
     (or `actions/attest-build-provenance` v4, which wraps it; hdf-libs uses
     that today). The tap fetches it from GitHub's attestation API.
   - **npm:** npm trusted publishing writes SLSA provenance. The tap fetches the
     bundle from the version's `dist.attestations.url` and adds
     `--bundle <file> --digest-alg sha512`. This was run against `@mitre/saf`
     1.8.0: it passed, and a wrong ref or a wrong commit was rejected.
   - **pypi, rubygems:** the source attests the exact file it uploads (the
     sdist, the `.gem`) with `actions/attest`, in its tag-triggered publish job.
     The tap verifies it like an archive. PyPI's own PEP 740 attestations would
     need a format conversion before `gh` can read them, and `gh` rejects
     RubyGems' own Sigstore bundles, so neither is the tap's input.
   - The source workflow that publishes must be triggered by the tag, so the
     attestation records `refs/tags/<tag>`. cis-bench and emasser change their
     publishing to meet this.

   Provenance covers the file the formula downloads. Dependencies resolved at
   install time are covered by Homebrew's own mechanisms instead: checksummed
   `resource` blocks (PyPI), a complete lock file installed by Bundler with
   `BUNDLE_COOLDOWN` (Ruby), and `--min-release-age` (npm).
4. **Updating the formula.** Before any tool runs, the bump compares the
   requested version with the formula on `main`. If they are equal it opens no
   PR. If a PR for that version is open, the write job re-checks the PR's head
   under Decision 5 and re-arms auto-merge only when those checks allow it.
   (`brew bump-formula-pr` exits 1 when the versions are equal,
   `bump-formula-pr.rb:406-411`.) Every `brew` command runs against the tap as
   installed by `Homebrew/actions/setup-homebrew`, names formulae as
   `mitre/tap/<name>` (`brew audit <path>` is disabled), and uses a checkout with
   full history (`fetch-depth: 0`) so `brew audit --git` can compare with
   `origin/HEAD`.
   - **archive kind: generated.** One generic renderer, a POSIX `sh` script that
     evaluates no formula code, produces the formula from the config (`desc`,
     `homepage`, license, binary name, archive name template, `platforms`, test
     assertion, caveats) and the verified checksums. `platforms` defaults to
     macOS arm64/x86_64 and Linux arm64/x86_64. The output has:
     - `on_macos`/`on_linux` blocks with nested `on_arm`/`on_intel` blocks, the
       documented form, so `brew readall --os=all --arch=all` can check that
       the formula loads on every platform (readall does not download URLs; the
       provenance step and the native install jobs cover those);
     - for a Linux-only tool, top-level `depends_on :linux`, and its two Linux
       URLs in both the `on_linux` and the `on_macos` blocks. A formula has to
       have a URL on every simulated platform: with none on macOS,
       `Formula#initialize` raises "formula requires at least a URL"
       (`formula.rb:321`). RuboCop's `ComponentsOrder` rejects `url` and
       `sha256` in a top-level `on_arm`/`on_intel` block, and
       `Hardware::CPU.arm?` reads the real CPU, not the simulated one, so
       neither of those forms works;
     - no `version` line;
     - a `livecheck` block with `url :stable` and `strategy :github_latest`
       (RuboCop's `LivecheckUrlProvided` requires the `url`), in the stanza
       order RuboCop's `ComponentsOrder` enforces;
     - the "generated — do not edit" header.

     Homebrew's bump commands cannot do this: they rewrite one top-level
     `url`/`sha256`, and these formulae have one URL per platform inside
     `on_*` blocks.
   - **registry kinds: checked-in formula, bumped by Homebrew's tools.** Each
     registry formula is written once by hand, following homebrew-core's shape
     for its ecosystem, and reviewed like any formula: `aws-cdk` for npm,
     `httpie` for PyPI, and Homebrew's documented Bundler pattern for Ruby
     (`docs/Language-Specific-Formulae.md`). On a release the bump changes only
     the version fields, with Homebrew's own code, the way homebrew-core's
     autobump does:
     - npm and rubygems: `brew bump-formula-pr --write-only --url <verified url>
       --sha256 <verified sha256> mitre/tap/<name>`, which writes locally and
       needs no GitHub credential;
     - pypi: `brew bump-formula-pr` cannot be used on release day. It always
       regenerates resources without the main-package cooldown exemption
       (`bump-formula-pr.rb:473`), so pip's `--uploaded-prior-to=P1D` hides the
       new release and the command exits. `update-python-resources` rewrites
       only `resource` blocks (`utils/pypi.rb:427-435`). The bump therefore
       rewrites the top-level `url` and `sha256` with Homebrew's formula AST
       (`Utils::AST::FormulaAST#replace_stable_stanza_value`, run through
       `brew ruby` after `Utils::GemSetup.install_bundler_gems!(groups:
       ["ast"])`, as Homebrew's own commands do; `brew ruby` does not install
       that gem group), then runs `brew update-python-resources
       --ignore-main-package-cooldown --version <version> mitre/tap/<name>`
       (plus `--install-dependencies` when the formula's `pypi_packages` names
       dependencies). A dependency still inside its own cooldown makes
       resolution fail before a PR exists; Decision 7 retries it.
     - A `.gem` is a plain tar of `metadata.gz`, `data.tar.gz` and
       `checksums.yaml.gz`, so Homebrew's tar unpacking leaves the gem's files
       packed, and `using: :nounzip` only copies the file
       (`unpack_strategy/uncompressed.rb:31`). The emasser formula downloads
       the attested `.gem` with `using: :nounzip`, unpacks it to reach its
       `Gemfile` and lock file, installs the dependencies with Bundler from
       that lock, and installs emasser itself from the `.gem` without
       resolving dependencies.

     A bump PR therefore changes only `url`, `sha256` and, for PyPI, resources.
     A change to a formula's body is an ordinary reviewed PR.
5. **Jobs and the write boundary.** The bump workflow has three jobs:
   - **verify** downloads the release files and checks digests and provenance
     (Decisions 2 and 3). It runs no formula code and no package build hooks.
     Its outputs are the verified version, tag commit, and each file's URL and
     sha256.
   - **update** produces the new formula (Decision 4). It holds no credential
     and is treated as running untrusted code: it evaluates formulae, and pip's
     resolution can run an sdist's build hooks. It hands its formula to the next
     job as an artifact.
   - **write** runs only trusted scripts from `main`, evaluates no formula, and
     takes its `url` and `sha256` values only from the verify job's outputs.
     Before minting the App token it refuses the artifact unless:
     - it changes exactly `Formula/<name>.rb`, as a regular file, and nothing
       else;
     - for an archive kind, the file equals what the write job renders itself
       from the config and the verify job's checksums;
     - for npm and rubygems, the only changed lines are the top-level `url` and
       `sha256`, and they equal the verify job's values;
     - for pypi, the same holds for the top-level `url` and `sha256`. Outside
       those two lines, the formula must equal `main` except for its resource
       stanzas. Every resource stanza in the new file, changed or not, must be
       exactly four lines: `resource "<name>" do`, a literal `url
       "https://files.pythonhosted.org/packages/…"`, a literal 64-hex `sha256`
       and `end`. No interpolation, backticks, semicolons or other calls are
       accepted, because a resource block is Ruby that Homebrew runs
       (`resource.rb:60`). For each resource, `<name>` is the stanza's
       resource name and `<version>` is parsed from the URL's file name, whose
       normalized distribution prefix must match `<name>`. PyPI's per-version
       JSON API (`/pypi/<name>/<version>/json`) must list that exact URL with
       that sha256, not yanked, uploaded at least a day earlier. A resource
       that keeps its name but points at another project fails here. Resources are installed
       from local files, so pip's own `--uploaded-prior-to` never sees them
       (`language/python.rb:434-439`); this check is their cooldown.
     - a genuine PyPI file can still be an unrelated package, and
       `virtualenv_install_with_resources` installs every resource
       (`language/python.rb:285-303`). So a pypi bump whose set of resource
       names differs from `main` is still opened as a PR, but without
       auto-merge: a maintainer reviews any added or removed dependency.

   Then the write job pushes a branch and opens a PR with a GitHub App
   installation token holding Contents and Pull requests write; a
   `GITHUB_TOKEN` PR would hold its checks for manual approval. Except as just
   described, it arms `gh pr merge --auto --merge --match-head-commit <sha>`.
   Every arm, including a re-arm on an already open PR, runs these checks on
   the PR's current head first. Homebrew's
   `brew bump` and `Homebrew/actions/bump-packages` are not used: they evaluate
   the formula and hold the write token in one process.
6. **Verification.** On every PR that changes `Formula/`:
   - one platform-independent job runs `brew audit --strict --online --git
     --os=all --arch=all` (with `--new` for an added formula) and
     `brew readall --os=all --arch=all`;
   - four platform jobs always run, with fixed check names (macOS Apple
     silicon, macOS Intel, Linux x86_64, Linux arm64). For each changed formula
     that supports the platform, each runs fatal steps against the PR's
     formula: `brew install`, `brew linkage --test` and `brew test`. A platform
     job with nothing to install passes without installing. Leaving a matrix
     leg out would leave its required check pending forever.

   `brew test-bot --only-tap-syntax` keeps running on PRs and pushes. No bottles
   are built or published.
7. **Release cooldown and retries.** Bumps open immediately; nothing waits a
   day. Homebrew's install-time cooldown (`RELEASE_COOLDOWN_DAYS = 1`;
   `std_npm_args` passes `--min-release-age=1`) applies to a formula's
   dependencies, so a release that pins a dependency published the same day
   fails `brew install` until that dependency is a day old. PyPI resource
   resolution has the same effect before a PR exists. An hourly scheduled job,
   using `GITHUB_TOKEN` with only `actions: write`, `contents: read` and
   `pull-requests: read`, re-runs:
   - the failed checks of open PRs on `bump/*` branches authored by the
     tap-publisher App;
   - failed bump workflow runs whose version is not yet on `main` and has no
     open PR (a re-run keeps the run's original inputs).

   It retries only runs and PRs created in the last 48 hours, by `created_at`;
   a re-run resets `run_started_at`, so that field cannot cap retries. A real
   failure fails again and stays visible.
8. **Merge gate and branch protection.**
   - A ruleset on `main` requires a pull request and every verification check.
     Its bypass list is empty, so no person, app or workflow pushes to `main`
     directly. Auto-merge merges when the checks pass.
   - A ruleset on every other branch lets only the tap-publisher App and the
     repository's maintainers (Maintain and Admin roles) create, update or
     delete branches. A tag ruleset blocks tag creation, update and deletion.
     The dispatch App needs Contents write to send `repository_dispatch`, which
     would otherwise let a compromised source project push to an open PR's
     branch.
   - Neither App has the Workflows permission, so neither can change a workflow
     file on any branch. The tap-publisher App's key is in an environment
     limited to `main`.

## Alternatives Considered

### Alternative A: Keep source-side direct push (status quo)
The source release renders and pushes to `main`. This is HashiCorp's tap model.
- **Pros:** Already works; one repository owns the whole release.
- **Cons:**
  - No install or test before users get the formula.
  - Every new tool copies the scripts.
  - Each source project holds a credential that can write `main`.
- **Why rejected:** Verification happens on users' machines.

### Alternative B: Homebrew's default flow — test-bot, bottles, `brew pr-pull`
What `brew tap-new` generates, and what docs.brew.sh describes for taps.
- **Pros:** Homebrew's standard machinery. Users can install without build tools
  (`formula_installer.rb` raises `UnbottledError` when there is no bottle and no
  development tools).
- **Cons:**
  - Bottles duplicate the release archives.
  - `pr-pull` pushes a bottle commit to `main`, which needs a ruleset bypass
    actor.
  - With no bottles `pr-pull` fails unless the PR is labelled `CI-no-bottles`.
  - `pr-automerge --publish` dispatches `publish-commit-bottles.yml`, not the
    generated `publish.yml`.
  - test-bot's failure-ignoring rule (Context) still applies until a bottle
    exists.
- **Why rejected:**
  - Verification depends on bottles we do not otherwise need.
  - The flow requires a direct-push actor.
  - The build-tools benefit is small: Homebrew's macOS installer sets up the
    Command Line Tools, and Homebrew on Linux expects build tools. It is larger
    for cis-bench and emasser, which compile native extensions; see Risks.

### Alternative C: test-bot `--only-formulae` without bottles
Run test-bot on the PR and discard any bottles.
- **Why rejected:**
  - `--build-from-source` skips the install and test steps entirely.
  - Without that flag, an unbottled existing formula's install and test failures
    are ignored. A bump PR can show green with a broken formula.

### Alternative D: Human merge after green checks (bufbuild/homebrew-buf)
- **Pros:** A person reads every bump.
- **Cons:** Every release waits on a person.
- **Why rejected:** A bump PR changes only generated or tool-written lines,
  enforced before the PR is opened (Decision 5), and the checks install and test
  it on every platform. Human review stays the path for formula bodies.

### Alternative E: Homebrew bump tooling as the whole pipeline
The candidates are `brew bump` / autobump, `Homebrew/actions/bump-packages`,
mislav/bump-homebrew-formula-action and dawidd6/action-homebrew-bump-formula.
- **Why rejected:**
  - For archive formulae, all of them rewrite one top-level `url`/`sha256`; a
    formula with per-platform URLs inside `on_*` blocks is out of their reach
    (Homebrew's `FormulaAST#stable_stanza` reads direct children only).
  - For registry formulae, `brew bump` and `bump-packages` open the PR from the
    process that evaluates the formula, and none of them verifies provenance.
    Decision 4 uses the commands and library they use underneath, inside the
    credential-free update job instead.

### Alternative F: Casks
Homebrew's policy for binary-only software in homebrew-core is casks, and
GoReleaser deprecated formula generation in favour of casks.
- **Why rejected:** Our tools ship Linux builds. That policy binds
  homebrew-core, not third-party taps.

### Alternative G: Generate registry formulae from config fragments
The config would carry each tool's install body, test and caveats as Ruby
fragments, and the renderer would emit the whole formula.
- **Why rejected:** Homebrew keeps registry formulae as checked-in Ruby and
  bumps only their version fields. Fragments put Ruby in strings, and RuboCop
  would see only the rendered output.

### Alternative H: Wait out the release cooldown before bumping
Hold a registry bump until the release is a day old, as `brew bump` does.
- **Why rejected:** Homebrew applies its cooldown to adopting third-party
  upstream releases ("Cooldowns on riskier ecosystems",
  `docs/Homebrew-Security-and-Supply-Chain.md`). Our own release is already
  bound to its tag and verified before merge, and users expect `brew upgrade`
  to see a release the day it ships. Decision 7 retries instead.

### Alternative I: Exempt cis-bench and emasser from tag-bound provenance
The tap would check only the registry digest for them.
- **Why rejected:** A registry digest proves only that the file did not change
  after upload, the gap this ADR exists to close. SLSA's verification guidance
  checks the source repository and ref
  (https://slsa.dev/spec/v1.0/verifying-artifacts).

### Alternative J: `rubygems/release-gem` for emasser's publishing
- **Why rejected:** It runs `bundle exec rake release`, which also creates and
  pushes git tags with a token it holds, and it turns on an attestation patch it
  marks experimental. The tag already exists in a tag-triggered job, and the
  tap's provenance comes from `actions/attest`. It also has no license file or
  statement. `rubygems/configure-rubygems-credentials`, the step it calls for
  trusted publishing, is used directly.

### Alternative K: Unpinned `gem install` for emasser
homebrew-core's `fastlane` runs `gem build` and `gem install`, which resolves
dependencies from rubygems.org at install time.
- **Why rejected:** Those dependencies have no pin and no cooldown. Homebrew's
  documentation prefers Bundler with a lock file that records the complete
  dependency set.

### Alternative L: Do nothing
- **Why rejected:** Five more tools would inherit unverified publishing and a
  copied render script.

## Prior Art

Health and licenses were read from each repository on 2026-11-18. None is
stale. We copy no code: ADOPT means we call the tool or action; REFERENCE means
design only.

| Candidate | License | Last release | Last merged PR | Last commit | Archived | Verdict |
|---|---|---|---|---|---|---|
| Homebrew/brew (`install`, `audit`, `linkage`, `test`, `readall`, `bump-formula-pr --write-only`, `update-python-resources`, `Utils::AST::FormulaAST`, test-bot tap syntax) | BSD-2-Clause | 7.0.14, 2026-11-16 | 2026-11-18 | 2026-11-18 | no | ADOPT |
| Homebrew/homebrew-core (`aws-cdk`, `httpie` formulae) | BSD-2-Clause | none | 2026-11-18 | 2026-11-18 | no | REFERENCE (registry formula bodies) |
| Homebrew/actions (`setup-homebrew`) | BSD-2-Clause | 2026.11.14.1, 2026-11-14 | 2026-11-13 | 2026-11-13 | no | ADOPT |
| Homebrew/actions (`bump-packages`) | BSD-2-Clause | same | same | same | no | REJECT (see E) |
| actions/create-github-app-token | MIT | v3.4.0, 2026-11-18 | 2026-11-18 | 2026-11-18 | no | ADOPT (already in use) |
| actions/attest | MIT | v4.2.2, 2026-08-04 | 2026-11-06 | 2026-11-06 | no | ADOPT (source side) |
| actions/attest-build-provenance | MIT | v4.2.2, 2026-08-06 | 2026-08-06 | 2026-08-21 | no | ACCEPT where already used (hdf-libs); new work uses `actions/attest`, as its README says |
| cli/cli (`gh attestation verify`, `gh pr create`, `gh pr merge --auto`, `gh run rerun`) | MIT | v2.105.0, 2026-11-18 | 2026-11-18 | 2026-11-18 | no | ADOPT |
| pypa/gh-action-pypi-publish | BSD-3-Clause | v1.14.3, 2026-10-20 | 2026-11-14 | 2026-11-14 | no | ADOPT (cis-bench source side; already in use) |
| rubygems/configure-rubygems-credentials | MIT | v2.1.0, 2026-06-12 | 2026-11-09 | 2026-11-09 | no | ADOPT (emasser trusted publishing) |
| rubygems/release-gem | no license file or statement found | v1.4.2, 2026-11-16 | 2026-11-16 | 2026-11-16 | no | REJECT (see J); `await-release.rb` informs the availability poll as design reference |
| trailofbits/pypi-attestations | Apache-2.0 | v0.0.31, 2026-11-02 | 2026-11-11 | 2026-11-11 | no | REJECT (the tap verifies `actions/attest` output instead) |
| goreleaser/goreleaser (`brews`, `homebrew_casks`) | MIT | v2.20.2, 2026-11-12 | 2026-11-18 | 2026-11-18 | no | REJECT (deprecated; casks macOS-oriented) |
| mislav/bump-homebrew-formula-action | Unlicense | v4.2, 2026-07-12 | 2026-08-14 | 2026-08-14 | no | REJECT (single URL; saf's current PAT-based bump) |
| dawidd6/action-homebrew-bump-formula | MIT | v11, 2026-11-17 | 2026-11-17 | 2026-11-17 | no | REJECT (wraps `bump-formula-pr`) |
| hashicorp/homebrew-tap (dispatch → template → push) | MPL-2.0 | none | 2026-11-13 | 2026-11-13 | no | REFERENCE (dispatch and render) |
| bufbuild/homebrew-buf (App-token PR, install-from-source CI) | Apache-2.0 | none | 2026-10-22 | 2026-10-22 | no | REFERENCE (PR and CI shape) |

## Consequences

**What becomes easier:**
- Each formula change is installed and tested on every platform it supports
  before it reaches `main`, and a failure leaves a red PR, not a broken formula.
- Onboarding a tool needs a config file, a tag-triggered attestation in its
  release (npm's provenance already counts) and a dispatch step. A registry
  tool also needs its formula written once.
- The tap's write path is one reviewed workflow, behind rulesets. Source
  projects hold only a dispatch credential.
- Users can verify provenance themselves with `gh attestation verify`.

**What becomes harder:**
- Hand-maintained or compiled formulae have no bottle path. A future formula
  that needs bottles reopens this decision in a new ADR.
- A source repository's release workflow must attest from a tag-triggered job
  before its formula can be bumped.
- Branches in the tap can be pushed only by maintainers and the tap-publisher
  App; other contributors open PRs from forks.

**Risks:**
- **Linux arm64 runner support.** It is unconfirmed whether
  `ghcr.io/homebrew/brew` runs on GitHub's Linux arm64 runners. The CI card
  proves it, or records native Homebrew on the runner as the fallback.
- **Flaky online audit.** `brew audit --online` makes GitHub API calls. A
  transient failure leaves the PR red; the hourly re-run (Decision 7) or a
  manual re-run recovers it.
- **Missing first live run.** v0.4.0 of claude-statusline has no attestations.
  The first positive run is a canary release (Phase 6b).
- **Same-day dependencies.** A registry release that pins a dependency
  published the same day reaches Homebrew up to a day later (Decision 7). saf
  pins its `@mitre/*` siblings exactly, so this is likely when they release
  together. The failure is inferred from `--min-release-age`, not yet run.
- **Install-time dependencies are outside provenance.** Provenance binds the
  file the formula downloads, not the dependencies npm, pip or Bundler resolve
  (Decision 3).
- **Native extensions.** cis-bench (for example lxml) and emasser (for example
  ffi) compile on the user's machine, which is slower and needs developer tools
  (`UnbottledError` otherwise). The exact list is confirmed on their cards.
- **Pinned runtimes.** emasser requires Ruby `~> 3.2` and core `ruby` is 4.0.9,
  so its formula depends on `ruby@3.4` until emasser supports Ruby 4.
- **emasser's lock file.** Its current `Gemfile.lock` lists only the
  `x64-mingw-ucrt` and `x86_64-linux` platforms and mixes development gems into
  the default set, so emasser has to produce a complete lock before its
  formula can be written (Phase 18a).
- **heimdall-cli runs as root.** It configures PostgreSQL, fapolicyd, firewalld
  and SELinux, and is invoked from `heimdall-server` RPM scriptlets. Homebrew's
  FAQ ("Why does Homebrew say sudo is bad?") warns that root-run code from a
  user-writable prefix trusts every account that can write that prefix. Its
  onboarding card decides how the formula and its docs handle that.
- **New-formula notability does not apply.** `brew audit --new` notability
  checks need repository data, and `FormulaAuditor#get_repo_data` returns early
  for any formula outside homebrew-core (`formula_auditor.rb:840`). They never
  run for mitre/tap.

## Owner decisions after review (2026-11-18)

1. **Separate dispatch App for source projects.** Source repositories get the
   key of a second App, installed on the tap with Contents write only, which
   sending `repository_dispatch` requires. They never hold the key of the
   App that opens and merges PRs; that key lives only in the tap. The rulesets
   (Decision 8) stop the dispatch App's token from pushing to any branch or
   tag.
   - Rejected: one shared App, because a source repository could then mint PR
     and Contents write.
   - Rejected: tap-side polling as the trigger, because it adds latency and
     drops the dispatch.
2. **Registry formats are in scope** (npm saf-cli, PyPI cis-bench, RubyGems
   emasser), with formulae that are never bumped by hand.
3. **hdf-cli joins the tap** on the archive path. It already attests its
   archives, so it needs no provenance change.
4. **cis-bench and emasser change their publishing** to tag-triggered jobs that
   attest the uploaded file; emasser also moves to RubyGems trusted publishing
   through `rubygems/configure-rubygems-credentials`.
5. **heimdall-cli joins later**, once it publishes tagged releases, as a
   Linux-only archive formula.
6. **Bump immediately**, with hourly re-runs of failed checks and failed bump
   runs.
7. **Registry formulae are checked in** and bumped only by Homebrew's own
   tools; "never bumped by hand" is how decision 2 is met.
8. **emasser's dependencies are pinned**: Bundler with a complete lock file.
9. **Branch protection with a maintainers bypass**: every branch except `main`
   can be written only by the tap-publisher App and repository maintainers.

## Implementation Plan

### Quality standards (every card inherits these)
- **Framework first:** verify with Homebrew's own commands (`brew install`,
  `brew audit --strict --online`, `brew linkage --test`, `brew test`,
  `brew readall --os=all --arch=all`, `test-bot --only-tap-syntax`), and bump
  registry formulae with Homebrew's own commands and library. Never use a
  test-bot mode whose failures are ignored.
- **Pinned actions:** every action is pinned to a full commit SHA. Checkouts use
  `persist-credentials: false`. Workflow permissions are least privilege per
  job.
- **Credentials:**
  - App tokens are minted per run with only the permissions that run needs.
  - No token is exposed to a job that executes formula code or resolves
    packages.
  - No personal tokens, and no long-lived registry API keys in source
    projects.
- **Untrusted input:** the dispatch payload, `workflow_dispatch` inputs, registry
  metadata, every downloaded file (`checksums.txt`, archives, packages,
  attestation bundles) and the update job's artifact are untrusted.
  - Inputs reach scripts only through environment variables, never through
    `${{ }}` expansion inside `run:`.
  - The formula name must match a config file, and the version must match
    `^[0-9]+\.[0-9]+\.[0-9]+$` (pre-releases such as `3.8.0-rc.1` are refused).
    Both are checked before any URL, tag or path is built from them.
  - Archive names come from the config's name template, never from
    `checksums.txt`. The checksums parser requires exactly the expected names,
    64 lowercase hex digits each, with no duplicates; names the config does not
    list (a monorepo release's other assets) are ignored.
  - Registry URLs are built from the config's package name and the validated
    version, and the metadata's tarball or file URL must match that registry's
    host and path pattern before it is fetched.
  - The write job accepts the update job's artifact only under Decision 5's
    checks.
  - Hostile-input fixtures test each rule: traversal names, a missing platform,
    a duplicate entry, a malformed hash, a non-semver version, a metadata URL
    on another host, and artifacts that change another file, change a
    formula's body, swap a verified `url` or `sha256`, put code inside a
    resource block, or add a genuine but unrelated PyPI package.
- **Scripts:** POSIX `sh` with `set -eu`. They fail loudly on every step, are
  safe to re-run, and are tested by running them against fixture inputs and
  comparing with golden output.
- **Docs:** README, `adopting.md` and `operations.md` change in the same card as
  the behaviour they describe.

### Shared abstractions (built first)
| Shared need | Used by | Card |
|---|---|---|
| Config format (`kind`, `tag_template`, `platforms`, registry fields) and archive renderer | bump workflow, every onboarding | Phases 2 and 10 |
| Verification workflow (audit job + four platform jobs) | every PR, every onboarding | Phase 3 |
| Verify job per kind | bump workflow | Phases 6a and 12 |
| Write job and its artifact checks | bump workflow | Phases 6b and 13b |

### Phases

Phases 1–9 deliver the flow for claude-statusline. Phases 10–19 add the other
kinds and onboard the remaining tools. Each phase or lettered sub-phase is one
card of story points 5 or less; a phase that spans two repositories is two
cards. Source-side cards go on the source repository's board.

#### Phase 1: Make the tap's CI run (unblocked — start here)
- **Files:** Modify `.github/workflows/tests.yml`.
- **Acceptance criteria:**
  - [ ] The workflow parses and its jobs start on push and on PR. The container
    `options` move into the matrix's `container` object, the form
    `Homebrew/brew#23154` fixed in the `tap-new` template.
  - [ ] `test-bot --only-tap-syntax` runs on every platform. It runs
    `brew audit --tap=mitre/tap`, so it is expected to fail on the current
    formula's redundant `version` line until Phase 2 re-renders it. That one
    known failure is recorded on the card; any other failure is fixed here.
- **Verification:** a PR run shows every job started, with the redundant
  `version` audit as the only failure. Phase 2 turns it green.

#### Phase 2: Generic archive renderer and config (blocked by Phase 1)
- **Files:**
  - Create the renderer script, the config file for claude-statusline, and
    fixture plus golden tests.
  - Modify `Formula/claude-statusline.rb` (re-rendered).
- **Acceptance criteria:**
  - [ ] Rendering from a `checksums.txt` produces the four-platform formula with
    nested `on_arm`/`on_intel`, no `version` line, and a `livecheck` block.
  - [ ] The renderer refuses a checksums file missing any configured platform.
  - [ ] The re-rendered claude-statusline formula passes `brew style`,
    `brew audit --strict --online --os=all --arch=all` and
    `brew readall --os=all --arch=all`, and `test-bot --only-tap-syntax` goes
    green.
  - [ ] It installs and passes `brew test` on this machine.
- **Verification:** the golden test, plus `brew audit --strict --online
  mitre/tap/claude-statusline`.

#### Phase 3: Verification workflow (blocked by Phase 1)
- **Files:**
  - Modify `.github/workflows/tests.yml`.
  - Delete `.github/workflows/publish.yml` and the bottle upload step.
- **Acceptance criteria:**
  - [ ] The workflow runs on every PR, with no path filter on the trigger. A PR
    that changes no formula skips the formula jobs, and a skipped required
    check counts as passed. A path filter on the trigger would leave required
    checks pending forever on other PRs.
  - [ ] Checkout uses `fetch-depth: 0`, the tap is set up with
    `Homebrew/actions/setup-homebrew`, and formulae are named
    `mitre/tap/<name>`.
  - [ ] One job runs `brew audit --strict --online --git --os=all --arch=all`
    (with `--new` for an added formula) and `brew readall --os=all --arch=all`.
  - [ ] Four platform jobs always run with fixed names. For each changed formula
    the platform supports, each runs `brew install` of the PR's formula (not
    `main`'s), `brew linkage --test` and `brew test`, and any failure fails it.
  - [ ] A deliberately broken formula on a PR turns the check red.
- **Verification:** two PR runs — the red one on the broken formula and a green
  one.

#### Phase 4: Repository settings, part 1 (blocked by Phase 3; owner actions)
- **Acceptance criteria:**
  - [ ] The tap-publisher App has Contents and Pull requests write, and nothing
    more.
  - [ ] The App's client ID and a private key are provisioned to the tap itself,
    as a variable and a secret of an environment limited to `main` that only
    the bump workflow uses.
  - [ ] A second "tap dispatch" App exists, installed on the tap with Contents
    write only. Its key is provisioned to each source project's release
    environment as that project onboards.
  - [ ] Auto-merge is enabled for the repository.
  - [ ] A `main` ruleset requires a pull request and the Phase 3 checks. The
    tap-publisher App is its only bypass actor, so today's direct push keeps
    working until the source side cuts over. Auto-merge needs the required
    checks: GitHub offers it only on a PR that cannot merge immediately.
    "Require branches to be up to date" is off; each bump PR touches only its
    own formula file, and the bump workflow serializes bumps per formula.
  - [ ] A ruleset on every branch except `main` limits create, update and
    delete to the tap-publisher App and the Maintain and Admin roles, and a
    tag ruleset blocks tag create, update and delete.
  - [ ] Each setting is read back through the API and pasted in the card.
- **Verification:**
  - `gh api repos/mitre/homebrew-tap/rulesets` and each ruleset by id;
  - `gh api repos/mitre/homebrew-tap --jq .allow_auto_merge`;
  - both Apps' permissions pages.

#### Phase 5: Source side A — attest release archives (claude-statusline board, blocked by Phase 4)
- **Files:** Modify `.github/workflows/release.yml`,
  `scripts/render-formula.sh` and its tests, and `docs/release-pipeline.md`;
  create the fallback workflow.
- **Acceptance criteria:**
  - [ ] A separate job after the release job downloads the four published
    archives and attests them with `actions/attest`, with `id-token: write` and
    `attestations: write`. Releases are immutable, so a transient attestation
    failure must be re-runnable without rebuilding, as the formula job is today.
  - [ ] The formula job leaves the tag-triggered release. It becomes a separate
    `workflow_dispatch` workflow in the `release` environment, which already
    holds its secrets, so it runs only when the owner starts it as a fallback.
    The environment admits only `v*` tags, so the fallback is started with
    `--ref v<version>`; the workflow must also exist on the default branch.
  - [ ] `render-formula.sh` stops writing the `version` line and writes the
    `livecheck` block, so a fallback push passes the tap's audit.
- **Verification:** the repository's gate passes; the attestations are checked
  on the canary release in Phase 6b.

#### Phase 6a: Bump workflow — verify job, archive kind (blocked by Phases 2, 3 and 5)
- **Files:** Create the bump workflow's trigger, input validation and verify
  job.
- **Acceptance criteria:**
  - [ ] It runs on `repository_dispatch` `formula-release` and on
    `workflow_dispatch`, and refuses an unknown formula or a non-semver version
    before using either.
  - [ ] The verify job checks each archive's sha256 and attestation with the
    Decision's flags, runs no formula code, and outputs the verified values.
  - [ ] Tests cover each refusal: no attestation (v0.4.0), an attestation from
    another tag or branch, and a checksum mismatch.
- **Verification:** a manual dispatch for v0.4.0 stops in the verify job.

#### Phase 6b: Bump workflow — update and write jobs, canary (blocked by Phase 6a)
- **Files:** Add the update and write jobs.
- **Acceptance criteria:**
  - [ ] The update job holds no credential. The write job renders the archive
    formula itself from the verify job's outputs and refuses an artifact that
    differs or touches another file.
  - [ ] The write job pushes a branch, opens the PR with an App token and arms
    auto-merge with `--match-head-commit`.
  - [ ] A version already on `main` opens no PR. A version with an open PR
    opens no other; auto-merge is re-armed on it only after the write job's
    checks pass on its current head.
  - [ ] Runs for the same formula are serialized with a concurrency group.
- **Verification (canary):** the owner cuts the first attested
  claude-statusline release after this card merges. Its attestations verify
  with the Decision's flags. A manual dispatch for it opens a PR, which passes
  the checks on four platforms and auto-merges. The Phase 5 fallback workflow
  is started only if the new path fails. This is the first positive live run.

#### Phase 7: Source side B — dispatch instead of push (claude-statusline board, blocked by Phase 6b)
- **Files:**
  - Modify `.github/workflows/release.yml`: send the dispatch after
    attestation succeeds.
  - Remove the fallback workflow, `scripts/render-formula.sh`,
    `scripts/publish-formula.sh`, their tests and make targets.
  - Modify `docs/release-pipeline.md`.
- **Acceptance criteria:**
  - [ ] The release sends the dispatch.
  - [ ] No source-side code writes the tap.
  - [ ] The next release runs end to end: dispatch, attestation check, PR,
    checks, auto-merge, `brew upgrade` installs it.

#### Phase 8: Repository settings, part 2 (blocked by Phase 7; owner action)
- **Acceptance criteria:**
  - [ ] The `main` ruleset's bypass list is empty.
  - [ ] The tap-publisher App's keys held by source projects are revoked and
    their environment secrets deleted. Source projects keep only the dispatch
    App's key.
  - [ ] A direct push to `main` is refused.
  - [ ] The rulesets are read back through the API and pasted in the card.

#### Phase 9: Documentation (with each phase; final pass after Phase 19)
- **Files:**
  - `README.md` ("How formulae get here");
  - `docs/adopting.md` (per kind: config, attestation, dispatch, and the
    one-time formula for registry kinds);
  - `docs/operations.md` (CI, rulesets, App permissions, hourly re-run, re-run
    runbook, contributing from forks);
  - `CLAUDE.md` (point to AGENTS.md).
- **Acceptance criteria:**
  - [ ] Docs describe the flow as built.
  - [ ] No document still teaches direct pushes.

#### Phase 10: Config kinds and Linux-only formulae (blocked by Phase 2)
- **Files:** Modify the renderer and config schema; add fixtures.
- **Acceptance criteria:**
  - [ ] Configs carry `kind`, `tag_template` and, for archives, `platforms`;
    unknown kinds and malformed templates are refused.
  - [ ] A Linux-only config renders top-level `depends_on :linux`, with the
    Linux URLs inside `on_arm`/`on_intel` in both `on_linux` and `on_macos`.
    The fixture formula passes `brew style`, loads on every simulated platform
    (`brew readall --os=all --arch=all`), and passes
    `brew audit --strict --online --os=all --arch=all`.
  - [ ] The bump workflow builds the tag from `tag_template`.
- **Verification:** the golden tests, plus style, readall and audit on the
  fixture formulae.

#### Phase 11: hdf-cli onboarding (two cards)
- **11a, tap (blocked by Phases 6b and 10):**
  - [ ] An archive config: binary `hdf`, archive template
    `hdf_{version}_{os}_{arch}.tar.gz`, tag `v{version}`, signer
    `.github/workflows/release.yml`.
  - [ ] A manual dispatch for hdf-cli 3.8.0 verifies, opens a PR, passes the
    checks and merges; the release's other assets are ignored.
  - **Verification:** the PR's checks and `brew install mitre/tap/hdf-cli`.
- **11b, hdf-libs (blocked by Phases 4 and 11a):**
  - [ ] The release workflow sends the dispatch after the release and its
    attestations exist, and not for `-rc` tags.
  - [ ] The next stable hdf-libs release bumps the formula through the
    dispatch.
  - **Verification:** that release's tap PR and its merge.

#### Phase 12: Registry verify job (blocked by Phases 6a and 10)
- **Files:** Registry fetchers for npm, PyPI and RubyGems, with fixtures.
- **Acceptance criteria:**
  - [ ] Exact-version lookup with a bounded poll; the run fails if the version
    does not appear.
  - [ ] The download's digest matches the registry metadata.
  - [ ] npm provenance is fetched from `dist.attestations.url` and verified with
    `--bundle --digest-alg sha512` and the Decision's flags; PyPI and RubyGems
    files are verified like archives.
  - [ ] Refusal tests: wrong tag, wrong commit, no attestation, digest
    mismatch, metadata URL on another host.
- **Verification:** the fixture tests, plus a live verify of `@mitre/saf`
  1.8.0.

#### Phase 13a: Registry update job (blocked by Phases 6b and 12)
- **Files:** Modify the bump workflow's update job.
- **Acceptance criteria:**
  - [ ] npm and rubygems formulae are updated with `brew bump-formula-pr
    --write-only`.
  - [ ] pypi formulae are updated with the AST rewrite (after installing
    Homebrew's `ast` gem group) and `update-python-resources
    --ignore-main-package-cooldown`.
- **Verification:** a fixture formula of each kind updated on a clean runner.

#### Phase 13b: Registry write checks (blocked by Phase 13a)
- **Files:** Modify the write job; add hostile fixtures.
- **Acceptance criteria:**
  - [ ] The write job enforces Decision 5's registry rules: verified top-level
    `url`/`sha256`, no other change for npm and rubygems, and for pypi the
    four-line stanza shape for every resource and each resource checked against
    PyPI's per-version JSON API (sha256, not yanked, at least a day old).
  - [ ] A pypi bump that adds or removes a resource name opens its PR without
    auto-merge.
  - [ ] Hostile fixtures are refused: a swapped `url` or `sha256`, an extra
    line in a body, a resource block containing a call or interpolation, a
    resource URL on another host, a resource less than a day old, and a
    resource that keeps its name but points at another PyPI project. A fixture
    that adds a genuine but unrelated PyPI package opens a PR with no
    auto-merge, and a repeated dispatch for that version leaves auto-merge
    off.
- **Verification:** the fixture tests.

#### Phase 13c: Hourly re-run (blocked by Phase 6b)
- **Files:** Create the hourly re-run workflow.
- **Acceptance criteria:**
  - [ ] It uses `GITHUB_TOKEN` with only `actions: write`, `contents: read` and
    `pull-requests: read`.
  - [ ] It re-runs failed checks on open `bump/*` PRs authored by the
    tap-publisher App, and failed bump runs whose version is not on `main` and
    has no open PR, created in the last 48 hours.
- **Verification:** a re-run observed on a deliberately failed bump run, and
  none on a run older than 48 hours.

#### Phase 14: saf-cli formula (tap; replaces g9x.1; blocked by Phases 13a and 13b)
- [ ] `Formula/saf-cli.rb` is written from homebrew-core's `aws-cdk` pattern
  (`std_npm_args`) at saf's previous release, through a reviewed PR.
- [ ] A manual dispatch for the current release bumps it through the pipeline.
- **Verification:** both PRs' checks, and `brew install mitre/tap/saf-cli`.

#### Phase 15: saf-cli source and migration (two cards; blocked by Phase 14)
- **15a, saf:**
  - [ ] The publish workflow dispatches after `npm publish` succeeds.
  - [ ] The mislav bump and `HOMEBREW_PAT` are removed.
  - **Verification:** the next saf release's tap PR.
- **15b, homebrew-saf:**
  - [ ] `tap_migrations.json` points `saf-cli` at `mitre/tap`.
  - **Verification:** `brew upgrade` on a machine with the old tap moves to
    `mitre/tap/saf-cli`.

#### Phase 16: cis-bench source (cis-bench board; blocked by Phase 4)
- [ ] The release tag is pushed by an identity whose pushes trigger workflows
  (a GitHub App token or a maintainer); tags pushed with `GITHUB_TOKEN` start no
  workflow.
- [ ] A workflow triggered by that tag builds the sdist, attests it with
  `actions/attest`, publishes with `pypa/gh-action-pypi-publish`, then
  dispatches.
- **Verification:** the release's sdist verifies with the Decision's flags.

#### Phase 17: cis-bench formula (tap; replaces g9x.4; blocked by Phases 13a, 13b and 16)
- [ ] The formula is written from homebrew-core's `httpie` pattern, with
  `exclude_packages` and native dependencies recorded.
- [ ] The release-day PyPI bump sequence runs on a live release.
- **Verification:** the bump PR's checks and `brew install mitre/tap/cis-bench`.

#### Phase 18: emasser (two cards)
- **18a, emasser (blocked by Phase 4):**
  - [ ] Trusted publishing through `rubygems/configure-rubygems-credentials`
    (a gem owner configures the trusted publisher on rubygems.org) replaces the
    API key and `actions/setup-ruby@v1`.
  - [ ] A complete lock file ships in the gem: darwin and Linux on arm64 and
    x86_64, without development gems in the default set.
  - [ ] A tag-triggered job builds the `.gem`, attests it with `actions/attest`,
    pushes it, then dispatches.
  - **Verification:** the release's `.gem` verifies with the Decision's flags,
    and `bundle install` from its lock succeeds on macOS and Linux.
- **18b, tap (replaces g9x.3; blocked by Phases 13a, 13b and 18a):**
  - [ ] The formula depends on `ruby@3.4`, downloads the attested `.gem` with
    `using: :nounzip`, unpacks it, installs the dependencies with Bundler from
    the gem's lock file (`GEM_HOME` in `libexec`, `BUNDLE_WITHOUT=development`),
    per Homebrew's Ruby guidance, and installs emasser itself from the `.gem`
    without resolving dependencies.
  - **Verification:** the bump PR's checks and `brew install mitre/tap/emasser`.

#### Phase 19: heimdall-cli (two cards; g9x.2)
- **19a, heimdall-cli (blocked by heimdall-cli's decision to publish releases):**
  - [ ] Tag-triggered releases with archives, `checksums.txt`, `actions/attest`
    and the dispatch.
  - **Verification:** a release whose archives verify with the Decision's
    flags.
- **19b, tap (blocked by Phases 6b, 10 and 19a):**
  - [ ] A Linux-only archive config, onboarded like hdf-cli.
  - [ ] The formula's caveats and the tap docs state how root-run commands are
    expected to be invoked, per the Risks entry.
  - **Verification:** the PR's checks and `brew install mitre/tap/heimdall-cli`
    on Linux.

### Ordering note
The `main` ruleset goes on in Phase 4 with the tap-publisher App as its only
bypass actor. That makes required checks and auto-merge work while the old
direct push can still publish. Phase 5 takes the old push out of the tag
release and keeps it as a manual fallback, so the canary release (Phase 6b) is
published by the new path. The source side switches to dispatch only after that
(Phase 7), and the bypass actor is removed last (Phase 8), so no release lands
while neither path works. The onboarding cards `homebrew-tap-g9x.1`–`.4` are
blocked by Phases 1–8 and rewritten as Phases 14, 19, 18 and 17; their current
text (saf 1.7.0, `test-bot --only-formulae`) is stale.

### Verification strategy (end to end)
- Run a manual dispatch against a release without attestations. It must stop.
- Put a broken formula on a PR. It must go red.
- Hand the write job artifacts that change another file, swap a verified value
  or put code in a resource block. It must refuse each.
- Run the claude-statusline canary release live. Expected results:
  - attestations verify;
  - the PR verifies on four platforms;
  - auto-merge lands it;
  - `brew upgrade` installs the new version, and `--version` reports it.
- Onboard hdf-cli and one registry tool through a live release each.
- Read the rulesets back and confirm that a direct push to `main`, and a push
  to any branch by the dispatch App, are refused.
