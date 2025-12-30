# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Claude Code plugin that provides a structured 7-phase workflow for bug investigation and fixing. The plugin orchestrates specialized agents to explore codebases, analyze potential root causes, form hypotheses, and validate fixes.

## Architecture

### Plugin Structure

```
.claude-plugin/
  plugin.json          # Plugin manifest (name, description, version)
agents/                # Specialized agent definitions
  bug-explorer.md      # Traces execution paths (model: sonnet)
  bug-investigator.md  # Analyzes code for bug patterns (model: opus)
  bug-historian.md     # Investigates git history (model: opus)
  bug-hypothesis.md    # Forms root cause hypotheses (model: opus)
  bug-fix-reviewer.md  # Reviews fix correctness (model: opus)
  bug-test-runner.md   # Runs related tests (model: sonnet)
commands/              # User-invocable slash commands
  bug-fix.md           # Main 7-phase workflow (/bug-fix)
  git-history.md       # Git history investigation (/git-history)
  test-check.md        # Test runner shortcut (/test-check)
```

### Agent Definition Format

Agents are markdown files with YAML frontmatter:

```yaml
---
name: agent-name
description: What triggers this agent
tools: Comma, Separated, Tool, List
model: sonnet|opus
color: yellow|blue|green|etc
---

System prompt content...
```

### Command Definition Format

Commands are markdown files with YAML frontmatter:

```yaml
---
description: Short description shown in /help
argument-hint: Placeholder text for arguments
---

Command prompt content with $ARGUMENTS placeholder...
```

## Workflow Phases

The `/bug-fix` command orchestrates a 7-phase workflow:

1. **Discovery** - Gather bug symptoms and reproduction steps
2. **Codebase Exploration** - Launch 2-3 bug-explorer agents in parallel
3. **Investigation & History** - Launch 3 bug-investigator + 1 bug-historian agents in parallel
4. **Hypothesis Formation** - Launch 3 bug-hypothesis agents with different focuses
5. **Fix Implementation** - Implement chosen fix approach (requires user approval)
6. **Fix Review & Test Validation** - Launch 3 bug-fix-reviewer + 1 bug-test-runner agents in parallel
7. **Summary** - Document root cause, fix, and prevention suggestions

## Key Design Patterns

- Agents run in parallel when possible for faster feedback
- Agents use confidence scores to prioritize findings
- Commands wait for user approval before making changes
- File:line references are used throughout for precision
- Key files identified in early phases are passed to later phases

## Version Management

- Version is stored in `.claude-plugin/plugin.json`
- CHANGELOG.md follows Keep a Changelog format
- Update both when releasing new versions

## Commit Convention

Use [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/) format:

```
<type>[optional scope]: <description>
```

Types: `feat` (new feature), `fix` (bug fix), `docs`, `style`, `refactor`, `perf`, `test`, `chore`, `ci`, `build`

Add `!` after type for breaking changes: `feat!: remove deprecated API`
