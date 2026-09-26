# Changelog

All notable changes to this project are documented here.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Changed

### Added


## [1.0.0] - 2026-09-23

Initial release.

### Added

- `LICENSE` (MIT), `CONTRIBUTING.md`, `CODE_OF_CONDUCT.md`, `SECURITY.md`, issue and pull
  request templates.
- Routes every `http`/`https` link to the most recently focused browser.
- Menu bar agent (`LSUIElement`) showing the live routing target and recent focus order.
- *Browsers* submenu to include or exclude browsers, with a guard that prevents the
  include set from ever becoming empty.
- *Fallback Browser* submenu for when no browser has been focused yet; the selection
  migrates automatically if that browser is excluded.
- *Set as Default Browser* and *Launch at Login*.
- `install.sh` one-liner installer: verifies the SHA-256 checksum and validates the
  bundle before replacing anything on disk.
