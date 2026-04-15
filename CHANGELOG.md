# Changelog

All notable changes to this project will be documented in this file.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).
This project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [1.1.0] - 2026-04-15

### Changed
- Action now accepts a pre-built `deb-path` input instead of building from `package-path` — the caller is responsible for building the `.deb`
- Default `deb-dir` changed from `deb` to `apt`
- README rewritten with full examples reflecting the new interface

### Removed
- `package-path` input and the internal `dpkg-deb --build` step

## [1.0.1] - 2026-04-13

### Fixed
- Prevent shell injection by passing action inputs as env vars instead of inline expressions
- Shorten action description to meet GitHub Marketplace 125-character limit
- Rename action to avoid GitHub Marketplace name conflict

### Added
- Versioning section in README

## [1.0.0] - 2026-04-13

### Added
- Initial release: build a `.deb` from a staged directory and publish it to an APT repository on GitHub Pages
- GPG signing support (`Release.gpg` and `InRelease`)
- Automatic retry on concurrent push conflicts
