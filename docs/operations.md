# Operating this tap

For the tap maintainer. Covers the publisher app, credential lifecycle,
CI behavior, and incident response. No secret values appear here or
anywhere in this repository — credentials live only as Actions secrets in
the consuming projects.

## The publisher app, by role

Automated formula pushes authenticate as a GitHub App owned by the MITRE
org (the "tap publisher"). Its security properties:

- **Installed on this repository only** — an installation token can touch
  nothing else in the org.
- **Permissions**: Contents read/write (to push formulas) and the implicit
  Metadata read. Nothing more.
- **Short-lived tokens**: consuming release workflows mint a 1-hour
  installation token per release via `actions/create-github-app-token`,
  explicitly requesting `permission-contents: write` only.
- Formula commits land authored as the app's bot identity — which is how
  you distinguish pipeline pushes from human ones in the history.

Audit the app's footprint at any time:

```sh
gh api apps/mitre-tap-publisher --jq '{slug, permissions}'
```

and review its installation targets under
*Org settings → GitHub Apps → the publisher app → Install App*.

## Provisioning a new consumer project

1. Confirm the project meets [adopting.md](adopting.md) (public repo,
   checksummed releases, pipeline-rendered formula).
2. In the app's settings page, generate a **new private key** for the
   consumer — one key per consuming project keeps revocation surgical.
3. In the consuming repo: create (or reuse) a deployment environment
   restricted to release-tag refs; add the key as an environment secret and
   the app's client ID as an environment variable. Never as plain repo
   secrets — those are readable by any branch's workflow.
4. Add the project's formula path to [CODEOWNERS](../.github/CODEOWNERS)
   and its row to the README table (via the project's first-formula PR).

## Key rotation

1. Generate a new private key in the app settings (old keys keep working —
   multiple keys can be active during the swap).
2. Update the environment secret in each consuming repository.
3. After every consumer has cut a green release on the new key (or you have
   verified the secret update directly with the consumer's maintainer),
   revoke the old key in the app settings.

Rotate on a calendar you can keep, and immediately on any suspicion of
exposure.

## Incident response

Suspected key exposure or a rogue formula push:

- **Revoke the affected private key** in the app settings — immediate; any
  outstanding installation token dies within its 1-hour lifetime.
- If needed, **suspend or uninstall the app** from this repository (org
  settings) — stops all automated pushes at once.
- Revert the bad formula commit on `main`; `brew test-bot` will verify the
  revert like any other change. Users who already installed a bad version
  recover with `brew reinstall` once the formula is fixed.
- Consumers' own releases are unaffected — this app cannot touch their
  repositories.

## CI in this repository

- **`tests.yml` (`brew test-bot`)** runs on every push to `main` and every
  PR, across macOS (Intel + Apple silicon images) and Linux: it builds each
  changed formula and runs its `test do` stanza. A pipeline-pushed formula
  that fails here is your signal to look at the source project's release.
- **`publish.yml` (`brew pr-pull`)** is the manual-dispatch path for
  hand-maintained formula PRs (bottle pulling); pipeline-generated formulas
  do not use it.

## Removing a formula

Delete `Formula/<tool>.rb` on `main`, drop its CODEOWNERS line and README
row, and — if the source project is retiring the channel rather than the
tool — leave a tombstone note in that project's README. If the tool moves
to another tap, add a `tap_migrations.json` entry so `brew` redirects
existing installs.
