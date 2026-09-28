# MITRE Homebrew Tap

The Homebrew tap for MITRE command-line tools.

## Formulae

| Formula | Description | Source |
|---|---|---|
| `claude-statusline` | Fast, configurable status line for Claude Code — single static Go binary | [mitre/claude-statusline](https://github.com/mitre/claude-statusline) |

## How do I install these formulae?

```sh
brew install mitre/tap/<formula>
```

Or tap once and install by name:

```sh
brew tap mitre/tap
brew install <formula>
```

Or, in a `brew bundle` `Brewfile`:

```ruby
tap "mitre/tap"
brew "<formula>"
```

## How formulae get here

- **Release automation**: projects using [goreleaser](https://goreleaser.com)
  publish their formula into `Formula/` on every tagged release. Do not edit
  generated formulae by hand — changes land via the next release of the
  source project.
- **Hand-maintained formulae**: open a pull request; `brew test-bot` builds
  and tests every formula change before merge.

Per-formula ownership is tracked in [CODEOWNERS](.github/CODEOWNERS).

## Documentation

- [Publishing your project's formula here](docs/adopting.md) — the adoption
  guide, with [mitre/claude-statusline](https://github.com/mitre/claude-statusline)
  as the reference implementation
- [Operating this tap](docs/operations.md) — publisher app, credential
  lifecycle, CI, incident response

For Homebrew itself: `brew help`, `man brew`, or
[Homebrew's documentation](https://docs.brew.sh).

## License

Licensed under the Apache License, Version 2.0 — see [LICENSE.md](LICENSE.md).

### NOTICE

© 2026 The MITRE Corporation. Approved for Public Release; Distribution
Unlimited. Case Number 18-3678.

See [NOTICE.md](NOTICE.md) for full terms.
