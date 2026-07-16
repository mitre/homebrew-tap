# Security Policy

## Reporting Security Issues

The MITRE SAF team takes security seriously. If you discover a security
vulnerability in this tap or one of its formulae, please report it
responsibly.

### Contact Information

- **Email**: [saf-security@mitre.org](mailto:saf-security@mitre.org)
- **GitHub**: use the repository's [Security tab](https://github.com/mitre/homebrew-tap/security) to report privately

### What to Include

1. **Description** of the vulnerability
2. **Steps to reproduce** the issue
3. **Potential impact** assessment
4. **Suggested fix** (if you have one)

### Response Timeline

- **Acknowledgment**: within 48 hours
- **Initial assessment**: within 7 days
- **Fix timeline**: varies by severity

## How Formulae Are Secured

- **Generated formulae** are published only by release automation in the
  source project's repository, committed by the tap's publishing app —
  never edited by hand
- **Checksums**: every binary a formula installs is pinned by its sha256
  from the source project's released `checksums.txt`
- **CI**: `brew test-bot` runs on every push and pull request
- **Vulnerabilities in a packaged tool** should be reported to that tool's
  own repository (see [CODEOWNERS](.github/CODEOWNERS) for ownership);
  issues in the tap's own files belong here
