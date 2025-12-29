# Bug Fix Plugin

A comprehensive, structured workflow for finding and fixing bugs with specialized agents for codebase exploration, root cause analysis, hypothesis formation, and fix validation.

## Overview

The Bug Fix Plugin provides a systematic 7-phase approach to debugging. Instead of jumping straight into code changes, it guides you through understanding the bug, exploring the codebase, investigating potential causes, forming hypotheses, and validating fixes—resulting in more accurate root cause identification and robust fixes.

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

## The 7-Phase Workflow

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

**Agents launched:**
- "Trace the execution path from [trigger] to [symptom]"
- "Map the error handling and edge cases in [area]"
- "Explore the data flow and state management in [component]"

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
```

### Phase 3: Investigation

**Goal**: Identify potential root causes through systematic analysis

**What happens:**
- Launches 2-3 `bug-investigator` agents in parallel
- Each analyzes for different bug patterns:
  - Null/undefined handling issues
  - Race conditions and timing issues
  - Logic errors and state management
- Consolidates findings with confidence scores
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

### Phase 6: Fix Review

**Goal**: Validate the fix doesn't introduce new issues

**What happens:**
- Launches 3 `bug-fix-reviewer` agents in parallel:
  - **Fix Correctness**: Does it address root cause?
  - **Regression Risk**: Could it break existing functionality?
  - **Edge Cases**: Are boundary conditions handled?
- Consolidates findings
- **Presents findings and asks what you want to do**:
  - Fix now
  - Fix later
  - Proceed as-is

**Example output:**
```
Fix Review Results:

Fix Assessment:
- Addresses Root Cause: Yes
- Overall Quality: Good
- Confidence Level: High

Potential Issues:

1. Missing Edge Case (Confidence: 82%)
   Location: src/services/saveService.ts:52
   Issue: Doesn't handle empty document case
   Impact: Could throw on new document save
   Suggested Fix: Add empty check before version compare

Side Effects:
- No breaking changes to existing callers
- Performance unchanged

What would you like to do?
```

### Phase 7: Summary

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

### `bug-fix-reviewer`

**Purpose**: Reviews fixes for correctness and potential issues

**Focus areas:**
- Fix correctness verification
- Regression risk assessment
- Side effect analysis
- Convention compliance

**When triggered:**
- Automatically in Phase 6
- Can be invoked manually after writing fixes

**Model**: Opus (thorough validation)

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

## Best Practices

1. **Provide reproduction steps**: The more specific, the faster the investigation
2. **Answer questions thoroughly**: Phase 1 sets the foundation for everything else
3. **Read the exploration findings**: Phase 2 reveals important context
4. **Choose fix approach deliberately**: Consider urgency vs long-term maintainability
5. **Don't skip review**: Phase 6 catches issues before they reach production

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

## Author

Dominik Drag

## Version

1.0.0
