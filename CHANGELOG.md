# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- `/bug-fix-tdd` command for TDD-style bug fixing workflow
  - 10-phase workflow with Red-Green-Refactor approach
  - **Phase 6 (Test Design)**: Design reproduction tests before writing
  - **Phase 7 (Red Phase)**: Write tests, verify they FAIL (proves bug exists)
  - **Phase 8 (Green Phase)**: Implement fix, verify tests PASS (proves fix works)
  - Automatic stop-and-ask when tests unexpectedly pass (bug can't be reproduced)
  - Separate state file (`claude-tmp/bug-fix-tdd-state.json`) and plan file (`claude-tmp/bug-fix-tdd-plan.md`)
  - Same configurable agent counts as `/bug-fix` (`--explorers`, `--investigators`, `--hypothesis`, `--reviewers`)

## [1.2.3] - 2026-01-09

### Added
- Configurable agent counts via command-line flags:
  - `--explorers=N` - Number of bug-explorer agents (default: 3, range: 1-10)
  - `--investigators=N` - Number of bug-investigator agents (default: 3, range: 1-5)
  - `--hypothesis=N` - Number of bug-hypothesis agents (default: 3, range: 1-5)
  - `--reviewers=N` - Number of bug-fix-reviewer agents (default: 3, range: 1-5)
- `allowed-tools` frontmatter to all commands for explicit tool restrictions

### Changed
- `bug-test-runner` agent: Changed model from sonnet to haiku for faster test execution

## [1.2.2] - 2026-01-09

### Added
- Phase 5: Planning - comprehensive fix plan creation before implementation
  - Consolidates information from Phases 1-4
  - Defines fix tasks (`FIX-NNN`), testing tasks (`TEST-NNN`), and review tasks (`REVIEW-NNN`)
  - Creates `claude-tmp/bug-fix-plan.md` with structured task breakdown
  - User approval gate via `AskUserQuestion` before implementation
  - Plan file tracks task completion across phases

### Changed
- Phase 6 (Implementation) now follows approved plan with `FIX-NNN` task tracking
- Phase 7 (Testing) now tracks `TEST-NNN` tasks in plan file
- Phase 8 (Review) now tracks `REVIEW-NNN` tasks in plan file
- PreCompact hook updated for 9-phase workflow and plan file management
- State file schema updated to include `planApproved` decision and `currentTask` tracking
- `bug-historian` agent: Changed model from opus to sonnet for faster execution
- `bug-test-runner` agent: Removed direct `Bash` tool access (now uses `BashOutput` only)

## [1.2.1] - 2026-01-08

### Removed
- PreToolUse auto-approval hook for state file operations
  - Simplified hook configuration
  - State file operations now follow standard permission flow

## [1.2.0] - 2026-01-07

### Changed
- Workflow expanded from 8 phases to 9 phases
- Various workflow improvements

## [1.1.4] - 2026-01-06

### Changed
- Moved state file from `.claude/` to `claude-tmp/` directory
  - `claude-tmp/bug-fix-state.json` - workflow state persistence
  - Enables auto-edit permission rules in `.claude/settings.local.json`
- Updated hooks to recognize new state file path

## [1.1.3] - 2026-01-05

### Fixed
- Corrected PreCompact hook structure to use nested `hooks` array wrapper (matching feature-dev pattern)

## [1.1.2] - 2026-01-04

### Changed
- Bug reproduction tests are now **mandatory** in Phase 6 (Testing)
  - Every bug fix must include at least one unit test that would have caught the bug
  - Tests must fail before the fix and pass after (proving the fix works)
  - Updated `bug-test-analyzer` agent to prioritize bug reproduction tests

## [1.1.1] - 2026-01-02

### Added
- Auto-approval hook for state file operations (Write, Edit, Bash)
  - State file modifications no longer require user permission
  - Enables seamless workflow state persistence during compaction
  - Positioned first in PreToolUse hook chain to intercept early

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
