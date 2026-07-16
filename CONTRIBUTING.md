# Contributing to the MITRE Homebrew Tap

Thank you for considering a contribution!

## Code of Conduct

By participating in this project, you are expected to uphold our [Code of Conduct](./CODE_OF_CONDUCT.md):

- Use welcoming and inclusive language
- Be respectful of differing viewpoints and experiences
- Gracefully accept constructive criticism
- Focus on what is best for the community
- Show empathy towards other community members

## How Can I Contribute?

### Reporting Problems

First decide where the problem lives:

- **A packaged tool misbehaves after a successful install** — report it to
  that tool's own repository (per-formula ownership is in
  [CODEOWNERS](.github/CODEOWNERS))
- **A formula fails to install, has a wrong checksum, or is out of date** —
  open an issue here with the full `brew install` output, your OS and
  architecture, and `brew config`

### Security Vulnerabilities

Do **not** open a public issue — see [SECURITY.md](./SECURITY.md).

## Formula Changes

- **Generated formulae** (published by a source project's release
  automation) must not be edited by hand — changes land via the next
  tagged release of the source project. If a generated formula is broken,
  open an issue so the source project can cut a fix.
- **Hand-maintained formulae**: open a pull request. Follow
  [Homebrew's formula cookbook](https://docs.brew.sh/Formula-Cookbook)
  and keep one formula change per PR.

## Pull Request Process

1. `brew test-bot` runs on every push and PR — it must be green before merge.
2. Verify locally where possible: `brew install --build-from-source <formula>`,
   `brew test <formula>`, `brew audit --strict <formula>`.
3. One logical change per PR.
