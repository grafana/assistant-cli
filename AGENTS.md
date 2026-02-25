# AGENTS.md

## Cursor Cloud specific instructions

This is a **documentation-only repository** for the Grafana Assistant CLI. It contains no source code, no build system, no automated tests, and no runnable application. The CLI binary is built from a separate internal Grafana repository and distributed via GitHub Releases, Homebrew, Docker, and PyPI.

### Repository structure

- `README.md` — main documentation and CLI reference
- `CHANGELOG.md` — release notes (auto-generated, synced from internal repo)
- `docs/` — detailed guides (Setup, Chat, Tunnel, Configuration, Docker, AGENTS-MD generation)
- `docs/images/` — screenshots
- `.github/CODEOWNERS` — code ownership
- `LICENSE` — Grafana Enterprise Plugin License

### Development workflow

The development workflow for this repo is editing Markdown files and committing them. Documentation is synced from the internal source repo via automated PRs.

### Linting

Install `markdownlint-cli2` globally to lint docs:

```bash
npm install -g markdownlint-cli2
markdownlint-cli2 "**/*.md"
```

Note: the existing docs have many pre-existing lint warnings (mostly MD013 line-length and MD024 duplicate headings in CHANGELOG.md). These originate from the upstream internal repo sync and should not be "fixed" in this repo.

### No build/test/run commands

There are no `build`, `test`, or `dev` commands. The README references `CONTRIBUTING.md` for development setup, but that file does not exist in this public repo (it lives in the internal source repo).
