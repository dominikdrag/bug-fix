# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [1.1.0] - 2026-01-02

### Added
- `bug-test-analyzer` agent (Opus) for proposing comprehensive test plans focused on regression prevention
- Phase 6: Testing - dedicated testing phase with user approval gate
  - Analyzer proposes test cases before writing
  - User must confirm via `AskUserQuestion` (proceed, modify scope, or skip)
  - Tests written directly to preserve fix context
  - Uses `bug-test-runner` for test execution
- Phase 7: Quality Review - user-driven fix selection with structured workflow
  - Waits for ALL review agents to complete before proceeding
  - Consolidates and deduplicates findings, organized by severity (Critical 90-100, Important 80-89)
  - Uses `AskUserQuestion` with `multiSelect: true` for per-issue selection
  - Applies only the fixes user explicitly selected
  - Offers re-review loop after fixes applied

### Changed
- Phase 6 split into two separate phases (Testing and Quality Review)
- `bug-fix-reviewer` output format standardized to match severity categories (Critical/Important)
- Phase numbering updated: previous Phase 7 (Summary) is now Phase 8
- State file now tracks 8 phases instead of 7
- State file schema updated to include `testStrategy` in decisions object
- PreCompact hook updated to reference 8-phase workflow

## [1.0.3] - 2026-01-02

### Added
- Workflow state persistence for compaction recovery
  - State file (`.claude/bug-fix-state.json`) tracks current phase, completed phases, and key decisions
  - `PreCompact` hook automatically saves state before conversation compaction
  - Workflow resumes from correct phase after compaction or session interruption
  - State file cleaned up on Phase 7 completion

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
