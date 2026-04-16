# Changelog

All notable changes to this project will be documented in this file.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).
This project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [2.0.0] - 2026-04-16

### 🎯 Major Architectural Change

Complete redesign to fix race conditions and GPG inconsistency issues through a **dual-mode dispatch architecture**.

### Added
- **Dispatch mode**: For app repos - uploads artifact and triggers pages repo
- **Publish mode**: For pages repo - receives dispatch and publishes package
- `mode` input for explicit mode selection (auto-detects if not specified)
- Sequential processing with `concurrency` group to prevent race conditions
- Artifact-based package transfer (1-day retention)
- Example workflows for both app and pages repos
- `ARCHITECTURE.md` documenting the complete solution
- New outputs: `mode`, `dispatched`

### Changed
- **BREAKING**: Action now requires two-step setup:
  1. Pages repo: workflow with `mode: publish`
  2. App repos: workflow with `mode: dispatch`
- **BREAKING**: Removed `pages-url` input (no longer needed)
- **BREAKING**: GPG secrets now stored only in pages repo (not app repos)
- Token requirement: `PAGES_REPO_TOKEN` only needs dispatch permission
- All inputs now optional with sensible defaults (except mode-specific required inputs)
- Repository checkout only happens in publish mode
- Improved error messages and validation

### Fixed
- **Race conditions**: Sequential queue prevents concurrent metadata corruption
- **GPG inconsistency**: Single centralized signing key eliminates state conflicts
- **Concurrency safety**: Eliminated retry loops and last-writer-wins scenarios
- **Security**: Reduced secret sprawl by centralizing GPG keys

### Removed
- **BREAKING**: Direct pages repo modification from app repos
- **BREAKING**: `pages-url` input (not needed with dispatch architecture)
- Retry logic (no longer needed - concurrency prevents conflicts)

### Migration Guide

See README section "Migration from v1" for complete migration steps.

**Quick summary:**
1. Add publish workflow to pages repo
2. Update app workflows to use dispatch mode
3. Move GPG secrets from app repos to pages repo
4. Remove `pages-url` from app workflows

---

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
