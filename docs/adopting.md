# Publishing your project's formula to this tap

How a MITRE project adds `brew install mitre/tap/<your-tool>` as a release
channel. The reference implementation is
[mitre/claude-statusline](https://github.com/mitre/claude-statusline) — its
[release-pipeline doc](https://github.com/mitre/claude-statusline/blob/main/docs/release-pipeline.md)
shows the whole chain working end to end.

## Prerequisites

- A **public** repository with tagged releases (release-asset URLs are not
  downloadable from a private repo, so the formula would not work).
- Per-platform release archives plus a `checksums.txt` of sha256 sums —
  goreleaser produces exactly this shape.
- A release pipeline that only publishes from verified commits (gate your
  release workflow on your test suite; see the reference implementation).

## The pattern

Your release workflow — never a human — writes `Formula/<your-tool>.rb`
into this tap. On each tagged release:

1. Publish the GitHub Release (archives + `checksums.txt`).
2. Render the formula **from the published `checksums.txt`** — per-platform
   `url` + `sha256` blocks, `desc`, `license`, and a `test do` stanza that
   asserts something real (the reference asserts `--version` reports the
   release version). Mark the file generated ("do not edit by hand") so
   drive-by PRs know where changes actually land.
3. Push it to this tap as a commit to `main`.

Rendering and pushing are ~two small POSIX scripts in the reference
implementation (`scripts/render-formula.sh`, `scripts/publish-formula.sh`) —
copy and adapt them, including their tests: the render script is
golden-tested, the publish script fails loudly on every step, exits 0 when
the formula is unchanged (safe re-runs), and renders only from a checksums
file you point it at. goreleaser's built-in `brews` publisher is deprecated
and `homebrew_casks` expects signed macOS binaries; the script pattern
avoids both.

## Credentials

Publishing is authenticated by the tap's **publisher GitHub App** (see
[operations.md](operations.md)) — no personal tokens. Ask the tap
maintainer (see [CODEOWNERS](../.github/CODEOWNERS)) to provision your
repository:

- the app's **client ID** as an Actions variable, and its **private key**
  as an Actions secret — both scoped to a deployment **environment** that
  only your release-tag refs can use, never plain repo secrets;
- your release workflow then mints a short-lived installation token per
  release with `actions/create-github-app-token`, requesting
  `permission-contents: write` and nothing else.

The default `GITHUB_TOKEN` cannot push cross-repo — the app token is the
supported mechanism.

## Checklist for your first formula

- [ ] Tap maintainer has provisioned the app credentials to your repo's
      release environment
- [ ] Release workflow renders from **published** checksums and pushes the
      formula as its own job (re-runnable without rebuilding artifacts)
- [ ] Formula has `desc`, `license`, per-platform `url`/`sha256`, and a
      meaningful `test do` stanza
- [ ] PR to this tap adding: your line to
      [`.github/CODEOWNERS`](../.github/CODEOWNERS) and your row to the
      README formulae table
- [ ] After your first release: `brew install mitre/tap/<your-tool>` on a
      real machine, and the installed tool reports the released version

This tap's `brew test-bot` CI fully builds and tests formulas on pull
requests; direct pushes to `main` (the pipeline path) get syntax checks
only — which is why the checklist's final step, a real `brew install`
after your release, is not optional: it is the build-level verification
for pipeline-published formulas.
