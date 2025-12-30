# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [1.0.2] - 2025-12-30

### Changed
- Standardized agent count in `/bug-fix` command to consistently use 3 bug-investigator and 3 bug-hypothesis agents (previously "2-3")

## [1.0.1] - 2025-12-29

### Added
- Bug-historian agent for investigating git history to find when and why bugs were introduced
- `/git-history` command for git history investigation
- Bug-test-runner agent for discovering and running tests related to bug fixes
- `/test-check` command for running tests related to code changes

## [1.0.0] - 2025-12-29

### Added
- Initial release of bug-fix plugin for Claude Code
- `/bug-fix` command for guided bug investigation and fixing workflow
- Bug-explorer agent for exploring codebase execution paths and data flow
- Bug-investigator agent for analyzing code patterns and potential root causes
- Bug-hypothesis agent for forming and validating hypotheses about bug causes
- Bug-fix-reviewer agent for reviewing proposed fixes
- Marketplace installation instructions
- Version field to plugin.json for spec compliance
- Acknowledgment for feature-dev plugin inspiration

### Infrastructure
- Added .gitignore and removed .DS_Store from tracking
