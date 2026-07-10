# Changelog

All notable changes to this project are documented here.

The format is loosely based on [Keep a Changelog](https://keepachangelog.com/).

## [Unreleased]

### Added
- GitHub Actions workflow (`.github/workflows/build.yml`) that builds the Windows
  x64 `v232 Launcher.exe` on a `windows-latest` runner. Enables producing the
  executable from any OS (incl. Apple Silicon Macs) without a local Windows
  toolchain. Manual-only triggers: `workflow_dispatch` and `v*` tag push; tag
  pushes also publish a GitHub Release with the `.exe` attached.
- Build instructions in `README.md` and `CLAUDE.md`.

### Changed
- `.gitignore` now excludes `bin/`, `obj/`, and `.claude/`; stopped tracking the
  stale `obj/` build intermediates.
