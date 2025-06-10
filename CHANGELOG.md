Changelog
=========

All notable changes to this project will be documented in this file.

## 0.2.0 - 2025-06-09

### Added
- Added support to easily extend with more processor templates

### Fixed
- Fixed the issue with the subprocess way of launching streams processes, which was flaky and caused issues in production.

### Changed
- Modified the basic architecture to use streams processor in place of the flaky subprocess way

## 0.1.0 - 2025-05-17
- Published the basic orchestrator for dynamic Benthos instances.