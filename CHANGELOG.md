# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## 0.0.18 (2026-02-24)


### Features

* **ci:** publish Python wheels to PyPI on release (#113)

## 0.0.17 (2026-02-23)


### Bug Fixes

* **ci:** create PR instead of pushing directly to homebrew repo (#110)

## 0.0.16 (2026-02-23)


### Bug Fixes

* **ci:** use `main` in grafana homebrew action (#108)

## 0.0.15 (2026-02-23)


### Features

* add interactive prompts and AGENTS.md generation following Claude's best practices (#104)
* **prompt:** add `--continue` flag support (#103)
* **release:** auto-publish Homebrew formula on release (#100)


### Bug Fixes

* **release:** place Homebrew cask in Casks/ directory (#102)

## 0.0.14 (2026-02-20)


### Bug Fixes

* **release:** sync `docs/` directory (#98)

## 0.0.13 (2026-02-20)


### Features

* **release:** add Homebrew formula generation (#89)


### Bug Fixes

* **release:** attach Homebrew cask to GitHub release (#90)
* **release:** set manifest version to prevent 1.0.0 bump (#95)
* **release:** strip internal repo links from changelog before public sync (#97)

## [Unreleased]

### Added

- Initial release of grafana-assistant CLI
- Interactive chat mode with SSE streaming support
- Single prompt mode for scripting
- Configurable API endpoints and authentication
- Cross-platform builds (Linux, macOS, Windows)
- Interactive tool approval flow in chat mode
  - Prompt user for approval before executing sensitive tools
  - Support for approve (y/Y) and deny (n/N/Esc) actions
  - Visual feedback showing approval status and tool information
