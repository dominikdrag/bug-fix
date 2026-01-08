# Bug Fix Plugin

A comprehensive, structured workflow for finding and fixing bugs with specialized agents for codebase exploration, root cause analysis, hypothesis formation, and fix validation.

## Overview

The Bug Fix Plugin provides a systematic 8-phase approach to debugging. Instead of jumping straight into code changes, it guides you through understanding the bug, exploring the codebase, investigating potential causes, forming hypotheses, implementing and testing fixes, and validating quality - resulting in more accurate root cause identification and robust fixes.

## Installation

Available via [dominikdrag-marketplace](https://github.com/dominikdrag/dominikdrag-marketplace). Run from Claude Code CLI:

```
/plugin marketplace add dominikdrag/dominikdrag-marketplace
/plugin install bug-fix@dominikdrag-marketplace
```

## Philosophy

Finding and fixing bugs effectively requires more than just changing code. You need to:
- **Understand the bug** thoroughly before exploring code
- **Explore the codebase** to understand execution paths
- **Investigate systematically** to identify potential causes
- **Form hypotheses** based on evidence, not assumptions
- **Validate fixes** to ensure they address root cause without regressions

This plugin embeds these practices into a structured workflow that runs automatically when you use the `/bug-fix` command.

## Command: `/bug-fix`

Launches a guided bug investigation and fixing workflow with 7 distinct phases.

**Usage:**
```bash
/bug-fix The login button sometimes shows an error after clicking
```

Or simply:
```bash
/bug-fix
```

The command will guide you through the entire process interactively.

## Command: `/test-check`

Quickly run tests related to your code changes without the full bug-fix workflow.

**Usage:**
```bash
# Test recent changes (uses git diff)
/test-check

# Test specific file
/test-check src/services/auth.ts

# Test multiple files
/test-check src/api/users.ts src/api/auth.ts
```

**What it does:**
- Detects your test framework (Jest, Vitest, Pytest, etc.)
- Finds tests related to the specified files
- Runs only relevant tests (not the full suite)
- Analyzes results and reports failures
- Identifies coverage gaps
- Suggests new tests if needed

**When to use:**
- After making any code changes
- Before committing to verify nothing broke
- When you want quick test feedback without full investigation

## Command: `/git-history`

Investigate git history to understand when and why code was written.

**Usage:**
```bash
# Investigate a specific file
/git-history src/services/auth.ts

# Investigate multiple files
/git-history src/api/users.ts src/utils/validate.ts

# No arguments - will ask what to investigate
/git-history
```

**What it does:**
- Uses git blame to find who last modified code
- Searches commit history for relevant changes
- Identifies the commit that introduced specific code
- Analyzes original developer intent from commit messages
- Finds related commits and previous fix attempts

**When to use:**
- When you need to understand why code was written a certain way
- To find when a bug was introduced
- To identify who to consult about specific code
- Before making changes to understand the history

## The 8-Phase Workflow

### Phase 1: Discovery

**Goal**: Understand the bug thoroughly

**What happens:**
- Gathers bug symptoms and reproduction steps
- Asks about expected vs actual behavior
- Identifies affected areas of the codebase
- Collects relevant error logs or stack traces
- Summarizes understanding and confirms with you

**Example:**
```
You: /bug-fix Users report occasional crashes when saving
Claude: Let me understand this bug...
        - What error message or behavior do users see?
        - How often does it happen? (always, sometimes, rarely)
        - What are the steps to reproduce?
        - When did this start happening?
```

### Phase 2: Codebase Exploration

**Goal**: Understand the code paths related to the bug

**What happens:**
- Launches 2-3 `bug-explorer` agents in parallel
- Each agent traces different aspects (execution flow, error handling, data flow)
- Agents return comprehensive analyses with key files to read
- Claude reads all identified files to build deep understanding
- Presents comprehensive summary of findings
- Saves key files list for historian in Phase 3

**Agents launched:**
- `bug-explorer`: "Trace the execution path from [trigger] to [symptom]"
- `bug-explorer`: "Map the error handling and edge cases in [area]"
- `bug-explorer`: "Explore the data flow and state management in [component]"

**Example output:**
```
Execution Flow:
- Entry point: src/components/SaveButton.tsx:45
- Handler calls: src/services/saveService.ts:23
- Data validation: src/utils/validate.ts:78
- API call: src/api/documents.ts:112

Key files to understand:
- src/services/saveService.ts - Main save orchestration
- src/api/documents.ts:112 - API error handling
- src/utils/validate.ts:78 - Validation logic

Areas of concern:
- saveService.ts:45 - Complex async logic
- documents.ts:120 - Error handling looks incomplete
```

### Phase 3: Investigation & History

**Goal**: Identify potential root causes AND understand when/why the code was written

**What happens:**
- Launches all agents **in parallel** for maximum efficiency:
  - 2-3 `bug-investigator` agents (code analysis)
  - 1 `bug-historian` agent (git history analysis)
- Investigators analyze for different bug patterns:
  - Null/undefined handling issues
  - Race conditions and timing issues
  - Logic errors and state management
- Historian analyzes git history of key files from Phase 2:
  - Uses git blame on suspicious code
  - Finds when problematic code was introduced
  - Understands original developer intent
- Consolidates findings with confidence scores and historical context
- Presents findings prioritized by likelihood

**Example output:**
```
High Confidence Findings:

1. Race Condition (Confidence: 85%)
   Location: src/services/saveService.ts:45
   Issue: State mutation during async save operation
   Evidence: State is read before await, written after
   Could cause: Intermittent data corruption on save

2. Missing Error Handling (Confidence: 78%)
   Location: src/api/documents.ts:120
   Issue: Network timeout not caught
   Evidence: Catch block only handles specific error types
   Could cause: Unhandled promise rejection

Historical Context:
- Culprit commit: abc1234 (2 weeks ago)
- Author: developer@example.com
- Message: "Optimize save performance by batching"
- The race condition was introduced when save was refactored
- Original code had proper locking, removed during optimization
```

### Phase 4: Hypothesis Formation

**Goal**: Form specific hypotheses and propose fix approaches

**What happens:**
- Launches 2-3 `bug-hypothesis` agents with different focuses
- Forms primary hypothesis based on evidence
- Notes alternative hypotheses if applicable
- Proposes three fix approaches with trade-offs:
  - **Quick Fix**: Minimal change, addresses symptom
  - **Proper Fix**: Addresses root cause appropriately
  - **Comprehensive Fix**: Fixes root cause + prevents recurrence
- Presents comparison with recommendation
- **Asks which approach you prefer**

**Example output:**
```
Primary Hypothesis:
Race condition in saveService.ts where state is read before
the async save completes, then overwritten with stale data.

Fix Approaches:

Quick Fix:
- Add null check before state write
- Prevents crash but doesn't fix race condition
- Low risk, 5 minutes to implement

Proper Fix:
- Use optimistic locking pattern
- Check state version before write
- Medium risk, addresses root cause

Comprehensive Fix:
- Refactor to use state machine pattern
- Add validation and guards
- Add tests for concurrent saves
- Higher effort but prevents entire class of bugs

Recommendation: Proper Fix - addresses root cause without
over-engineering. Quick fix would need revisiting.

Which approach would you like to use?
```

### Phase 5: Implementation

**Goal**: Implement the chosen fix

**What happens:**
- **Waits for explicit approval** before starting
- Reads all relevant files identified in previous phases
- Implements fix following codebase conventions
- Makes minimal changes (no unrelated refactoring)
- Updates todos as progress is made

**Notes:**
- Implementation only starts after you approve
- Follows patterns discovered in Phase 2
- Uses fix approach chosen in Phase 4
- Explains changes made after completion

### Phase 6: Testing

**Goal**: Ensure the fix is properly tested with user-approved strategy

**Critical Requirement**: Every bug fix **must** include at least one unit test that would have caught the bug. This test should fail before the fix and pass after.

**What happens:**

1. **Test Analysis**: Launches `bug-test-analyzer` agent to:
   - Identify the exact conditions that trigger the bug
   - Propose test cases that specifically reproduce the bug scenario
   - Identify edge cases related to the bug
   - Note mocking requirements and special setup

2. **User Approval**: Presents test plan and asks for explicit confirmation:
   - "Proceed with proposed testing strategy"
   - "Modify testing scope" (describe changes)
   - "Skip testing phase" (only if testing is truly not feasible)

3. **Test Writing**: If approved, writes tests directly:
   - **Bug reproduction test (REQUIRED)**: Exercises the exact scenario that caused the bug
   - Edge case tests: Cover related edge cases exposed by the bug
   - Regression prevention tests: Guard against similar issues

4. **Test Execution**: Launches `bug-test-runner` to run tests and report results

5. **Validation**: Fixes any failing tests and re-runs until all pass

**Example output:**
```
Test Plan for Null Pointer Fix:

Project Test Patterns:
- Framework: Jest
- Location: __tests__/ co-located with source
- Naming: *.test.ts

Tests Needed:

Regression Prevention (High Priority):
1. Test null input handling
   - Verifies original bug scenario is fixed
   - Input: null user object
   - Expected: Returns default user without crash

2. Test undefined input handling
   - Edge case related to bug
   - Input: undefined user object
   - Expected: Returns default user without crash

Related Scenarios (Medium Priority):
3. Test empty object handling
   - Input: {} empty object
   - Expected: Returns user with default values

Do you want to proceed with this testing strategy?
```

### Phase 7: Quality Review

**Goal**: Ensure fix quality and correctness through user-reviewed findings

**What happens:**

1. **Parallel Review**: Launches 3x `bug-fix-reviewer` agents:
   - **Agent 1**: Fix correctness (does it address root cause?)
   - **Agent 2**: Regression risk (could it break existing functionality?)
   - **Agent 3**: Edge cases and side effects

2. **Consolidation**: Waits for all agents, then organizes findings by severity:
   - **Critical Issues** (Confidence 90-100): Must be fixed
   - **Important Issues** (Confidence 80-89): Should be addressed
   - **Suggestions** (Confidence <80): Nice to have

3. **User Selection**: Presents findings and asks which issues to fix (multi-select):
   - Each issue is selectable
   - User chooses which to address
   - "Skip all - proceed to summary" option available

4. **Apply Fixes**: Only applies the fixes the user selected

5. **Re-review Option**: If fixes applied, asks whether to run review again

**Example output:**
```
Review Findings Summary:

Critical Issues (1):
1. [src/auth.ts:45] Missing null check after user lookup - 95%
   Impact: Could cause crash if user not found
   Fix: Add null check before accessing user.id

Important Issues (2):
1. [src/auth.ts:52] Edge case not handled - 85%
   Impact: Unexpected behavior for empty username
   Fix: Add empty string validation

2. [src/auth.ts:67] Potential memory leak - 82%
   Impact: Event listener not cleaned up
   Fix: Add cleanup in finally block

Which issues would you like to fix?
```

### Phase 8: Summary

**Goal**: Document what was found and fixed

**What happens:**
- Marks all todos complete
- Summarizes:
  - Root cause identified
  - Fix applied with rationale
  - Files modified
  - Verification steps
  - Prevention suggestions
  - Recommended next steps

**Example:**
```
Bug Fix Complete: Intermittent Save Crashes

Root Cause:
Race condition in saveService.ts where concurrent saves
could overwrite each other due to stale state reads.

Fix Applied:
Implemented optimistic locking with version checking.
State now validates version before write, retries on conflict.

Files Modified:
- src/services/saveService.ts
- src/types/document.ts
- src/api/documents.ts

Verification:
- Test with rapid save button clicks
- Test with slow network (throttle to 3G)
- Verify saves complete in order

Prevention:
- Consider adding state machine for complex state flows
- Add concurrent access tests to CI
```

## Agents

### `bug-explorer`

**Purpose**: Explores codebase to understand code paths related to bugs

**Focus areas:**
- Entry points and execution flow
- Data flow and transformations
- Error handling paths
- Related components and dependencies

**When triggered:**
- Automatically in Phase 2
- Can be invoked manually when exploring bug-related code

**Model**: Sonnet (fast exploration)

### `bug-historian`

**Purpose**: Uses git history to find when and why bugs were introduced

**Focus areas:**
- Git blame analysis on suspicious code
- Commit history search
- Finding the culprit commit
- Understanding original developer intent
- Identifying previous fix attempts

**When triggered:**
- Automatically in Phase 3 (alongside bug-investigator, after key files identified)
- Can be invoked manually via `/git-history` command
- Can be invoked manually to investigate code history

**Model**: Opus (deep analysis of intent and history)

### `bug-investigator`

**Purpose**: Analyzes code for common bug patterns and potential root causes

**Focus areas:**
- Null/undefined handling issues
- Race conditions and timing
- Logic errors and state management
- Error handling gaps
- Resource leaks

**When triggered:**
- Automatically in Phase 3
- Can be invoked manually for deep analysis

**Model**: Opus (deep analysis)

### `bug-hypothesis`

**Purpose**: Forms and validates hypotheses about root causes

**Focus areas:**
- Evidence synthesis
- Hypothesis formation and validation
- Fix approach proposals
- Trade-off analysis

**When triggered:**
- Automatically in Phase 4
- Can be invoked manually for hypothesis generation

**Model**: Opus (deep reasoning)

### `bug-test-analyzer`

**Purpose**: Analyzes bug fixes and proposes comprehensive test plans focused on regression prevention

**Focus areas:**
- Bug and fix context analysis
- Project test pattern discovery
- Regression prevention test proposals
- Edge case identification
- Test prioritization

**When triggered:**
- Automatically in Phase 6 (Testing)
- Can be invoked manually when planning tests for bug fixes

**Model**: Opus (advanced reasoning for test planning)

### `bug-fix-reviewer`

**Purpose**: Reviews fixes for correctness and potential issues

**Focus areas:**
- Fix correctness verification
- Regression risk assessment
- Side effect analysis
- Convention compliance

**When triggered:**
- Automatically in Phase 7 (Quality Review)
- Can be invoked manually after writing fixes

**Model**: Opus (thorough validation)

### `bug-test-runner`

**Purpose**: Validates fixes through automated testing

**Focus areas:**
- Test framework detection
- Related test discovery
- Test execution and result analysis
- Coverage gap identification
- New test suggestions

**When triggered:**
- Automatically in Phase 6 (Testing)
- Can be invoked manually to run tests for any code changes
- Can be invoked manually via `/test-check` command

**Model**: Sonnet (fast execution)

## Usage Patterns

### Full workflow (recommended for complex bugs):
```bash
/bug-fix The API sometimes returns stale data after updates
```

Let the workflow guide you through all 7 phases.

### Manual agent invocation:

**Explore bug-related code:**
```
"Launch bug-explorer to trace how the caching system works"
```

**Investigate git history:**
```
"Launch bug-historian to find when src/services/auth.ts was last modified and why"
```

**Investigate potential causes:**
```
"Launch bug-investigator to analyze the authentication flow for race conditions"
```

**Form hypothesis:**
```
"Launch bug-hypothesis to synthesize findings and propose fixes"
```

**Review a fix:**
```
"Launch bug-fix-reviewer to check my recent changes"
```

**Run tests for changes:**
```
"Launch bug-test-runner to find and run tests for src/services/auth.ts"
```

## Best Practices

1. **Provide reproduction steps**: The more specific, the faster the investigation
2. **Answer questions thoroughly**: Phase 1 sets the foundation for everything else
3. **Read the exploration findings**: Phase 2 reveals important context
4. **Choose fix approach deliberately**: Consider urgency vs long-term maintainability
5. **Don't skip testing and review**: Phases 6-7 catch issues before they reach production

## When to Use This Plugin

**Use for:**
- Bugs that are hard to reproduce
- Intermittent or timing-related issues
- Bugs in unfamiliar code
- Complex bugs spanning multiple files
- When you want systematic investigation

**Don't use for:**
- Obvious typos or syntax errors
- Simple, single-line fixes
- Well-understood bugs with known fixes
- Urgent hotfixes with no time for investigation

## Requirements

- Claude Code installed
- Git repository (for code exploration)
- Project with existing codebase

## Troubleshooting

### Investigation takes too long

**Issue**: Agents are slow to return findings

**Solution**:
- Provide more specific bug description
- Narrow down the affected area
- Agents run in parallel when possible

### Too many hypotheses

**Issue**: Phase 4 presents too many possibilities

**Solution**:
- Provide more specific reproduction steps
- Focus the investigation scope
- Trust the primary hypothesis recommendation

### Fix doesn't work

**Issue**: Bug still occurs after fix

**Solution**:
- Return to Phase 3 with new information
- The hypothesis might have been incorrect
- Additional root causes may exist

## Acknowledgments

This plugin is based on the [feature-dev plugin](https://github.com/anthropics/claude-code/tree/main/plugins/feature-dev) from Anthropic's official Claude Code repository.

## Author

Dominik Drag

## Version

1.2.1
