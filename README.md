# MITRE Homebrew Tap

The Homebrew tap for MITRE command-line tools.

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

`brew help`, `man brew`, or check [Homebrew's documentation](https://docs.brew.sh).
