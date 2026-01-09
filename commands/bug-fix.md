---
description: Guided bug investigation and fixing workflow with codebase exploration and root cause analysis
argument-hint: Optional bug description or symptoms
---

# Bug Fix Workflow

You are helping a developer find and fix a bug or defect in their code. Follow a systematic approach: understand the bug thoroughly, explore the codebase, investigate potential causes, form hypotheses, and implement a fix with proper validation.

---

## Workflow State Management

This workflow uses a state file (`claude-tmp/bug-fix-state.json`) to persist progress across conversation compaction and session interruptions.

### On Workflow Start

**FIRST**, check if `claude-tmp/bug-fix-state.json` exists:
- **If file exists and `active: true`**: This is a RESUMED workflow
  - Read the state file to understand current progress
  - Inform the user: "Resuming bug-fix workflow from Phase {currentPhase}"
  - Display completed phases and key decisions from the state
  - Continue from the current phase (do NOT restart from Phase 1)
- **If file does not exist**: This is a NEW workflow
  - Create initial state file with `active: true, currentPhase: 1, completedPhases: []`
  - Proceed with Phase 1

### State File Format

```json
{
  "active": true,
  "currentPhase": 1,
  "completedPhases": [],
  "bugDescription": "...",
  "decisions": {
    "fixApproach": null,
    "planApproved": null,
    "testStrategy": null
  },
  "summary": "Brief context for resumption"
}
```

### Updating State

At the START of each phase, update the state file:
- Set `currentPhase` to the new phase number
- Update `summary` with relevant context

At the END of each phase, update the state file:
- Add the phase number to `completedPhases`
- Store any decisions made (fix approach selection, plan approval, test strategy approval)

---

## Core Principles

- **Gather information first**: Understand the bug symptoms, reproduction steps, and expected behavior before exploring code
- **Understand before acting**: Read and comprehend existing code patterns and execution paths first
- **Read files identified by agents**: When launching agents, ask them to return lists of the most important files to read. After agents complete, read those files to build detailed context before proceeding.
- **Form hypotheses**: Don't jump to conclusions - form testable hypotheses based on evidence
- **Validate fixes thoroughly**: Ensure fixes address root cause, not just symptoms
- **Use TodoWrite**: Track all progress throughout

---

## Phase 1: Discovery

**Goal**: Understand the bug or defect thoroughly

Initial request: $ARGUMENTS

**Actions**:
1. Create todo list with all phases
2. Gather bug information from user:
   - What are the symptoms? (error messages, unexpected behavior, crashes)
   - How do you reproduce it? (steps, inputs, conditions)
   - When did it start? (recent change, always existed, intermittent)
   - What is the expected behavior vs actual behavior?
   - What area of the codebase is affected?
   - Any error logs, stack traces, or relevant output?
3. Summarize understanding and confirm with user

**Critical**: Get clear reproduction steps before proceeding. If the user doesn't have them, help them identify what triggers the bug.

---

## Phase 2: Codebase Exploration

**Goal**: Understand the code paths related to the bug

**Actions**:
1. Launch 2-3 bug-explorer agents in parallel. Each agent should:
   - Trace through the code comprehensively from trigger point to symptom
   - Target a different aspect (e.g., entry points and flow, error handling paths, related features)
   - Include a list of 5-10 key files to read

   **Example agent prompts**:
   - "Trace the execution path from [trigger] to [symptom], identifying all code involved"
   - "Map the error handling and edge cases in [affected area]"
   - "Find similar code patterns and how they handle [situation]"
   - "Explore the data flow and state management in [component]"

2. Once the agents return, read all files identified by agents to build deep understanding
3. Present comprehensive summary of findings:
   - Execution flow with file:line references
   - Key components involved
   - Potential areas of concern identified
4. **Save the list of key files** for the historian agent in Phase 3

---

## Phase 3: Investigation & History

**Goal**: Identify potential root causes AND understand when/why the code was written

**CRITICAL**: This is where deep analysis happens. DO NOT SKIP.

**Actions**:
1. Launch all investigation agents **in parallel** (3 bug-investigator + 1 bug-historian):

   **Bug Investigator Agents (3x)** - analyze the code for issues:
   - Common bug patterns (null handling, race conditions, logic errors)
   - State management and data flow issues
   - Error handling gaps and edge cases

   **Example investigator prompts**:
   - "Analyze [code area] for null/undefined handling issues, race conditions, and logic errors"
   - "Examine state management in [component] for potential issues"
   - "Check error handling completeness in [execution path]"

   **Bug Historian Agent (1x)** - analyze git history of key files from Phase 2:
   - Use git blame on the specific files/lines identified as suspicious
   - Find when the problematic code was introduced
   - Understand the original intent of the code
   - Look for previous attempts to fix similar issues

   **Example historian prompt**:
   - "Investigate git history of [key files from Phase 2] to find when and why the suspicious code was written, and identify the commit that may have introduced the bug"

2. Consolidate findings from all agents
3. Present findings with confidence scores:
   - Potential issues found with file:line references
   - Evidence supporting each finding
   - Prioritized by confidence level
   - **Historical context**: Culprit commit, original intent, previous fix attempts

---

## Phase 4: Hypothesis Formation

**Goal**: Form specific hypotheses about root cause and propose fix approaches

**Actions**:
1. Launch 3 bug-hypothesis agents in parallel with different focuses:
   - Most likely hypothesis based on evidence
   - Alternative hypothesis considering edge cases
   - Comprehensive analysis considering systemic issues

   **Each agent should propose a fix approach**:
   - **Quick Fix**: Minimal change, addresses symptom directly
   - **Proper Fix**: Addresses root cause with appropriate refactoring
   - **Comprehensive Fix**: Fixes root cause + adds guards/tests to prevent recurrence

2. Review all hypotheses and form your opinion on which is most likely correct
3. Present to user:
   - Primary hypothesis with supporting evidence
   - Alternative hypotheses if applicable
   - Fix approaches with trade-offs comparison
   - **Your recommendation with reasoning**
4. **Ask user which approach they prefer**

---

## Phase 5: Planning

**Goal**: Create a comprehensive fix plan before implementation begins

**Actions**:
1. **Consolidate Information**: Gather all outputs from Phases 1-4:
   - Bug symptoms and reproduction steps (Discovery)
   - Codebase context, execution paths, and key files (Exploration)
   - Investigation findings with confidence scores (Investigation)
   - Historical context and culprit commits (History)
   - Selected hypothesis and fix approach (Hypothesis Formation)

2. **Define Fix Tasks**: Break down the chosen fix approach into discrete, actionable tasks:
   - Create tasks for each file to be modified
   - Establish task dependencies where needed
   - Assign task IDs: `FIX-NNN` for fix implementation, `TEST-NNN` for testing, `REVIEW-NNN` for quality

   **Phase Scope Rules** (document explicitly in plan):
   - **Phase 6 (Implementation)**: Works ONLY on `FIX-NNN` tasks
   - **Phase 7 (Testing)**: Works ONLY on `TEST-NNN` tasks
   - **Phase 8 (Review)**: Works ONLY on `REVIEW-NNN` tasks

3. **Define Acceptance Criteria**: Extract or derive acceptance criteria from the fix approach
   - Each criterion should be testable/verifiable
   - Assign IDs: `AC-NNN`
   - Must include: "Bug no longer reproduces under original conditions"

4. **Write Plan File**: Create `claude-tmp/bug-fix-plan.md` with this structure:
   ```markdown
   # Bug Fix Plan

   > **Status**: In Progress
   > **Current Task**: [none]
   > **Created**: [ISO timestamp]
   > **Last Updated**: [ISO timestamp]

   ## Bug Summary
   [Summary from Phase 1]

   ## Root Cause Analysis
   [Selected hypothesis and supporting evidence from Phase 4]

   ## Codebase Context
   [Key files and execution paths from Phases 2-3]

   ## Historical Context
   [Relevant git history findings from Phase 3]

   ## Selected Fix Approach
   [Quick/Proper/Comprehensive fix with rationale]

   ## Phase Scope Rules
   - **Phase 6 (Implementation)**: Works ONLY on `FIX-NNN` tasks
   - **Phase 7 (Testing)**: Works ONLY on `TEST-NNN` tasks
   - **Phase 8 (Review)**: Works ONLY on `REVIEW-NNN` tasks

   ## Fix Tasks
   - [ ] **FIX-001**: [description]
     - Files: `path/to/file.ts:line`
     - Depends on: none
   - [ ] **FIX-002**: [description]
     - Files: `path/to/file.ts`
     - Depends on: FIX-001

   ## Testing Tasks
   - [ ] **TEST-001**: Bug reproduction test
   - [ ] **TEST-002**: [edge case description]

   ## Quality Tasks
   - [ ] **REVIEW-001**: Run code reviewers
   - [ ] **REVIEW-002**: Apply selected fixes

   ## Acceptance Criteria
   - [ ] **AC-001**: Bug no longer reproduces under original conditions
   - [ ] **AC-002**: [additional criterion]

   ## Progress Log
   | Timestamp | Event |
   |-----------|-------|
   | [ISO] | Planning phase completed |
   ```

5. **Present Full Plan**: Display the complete plan content to the user by reading and showing `claude-tmp/bug-fix-plan.md`

**IMPORTANT**: Present the FULL plan to the user - do NOT summarize or condense. The user needs complete visibility into every task, its dependencies, and acceptance criteria to make an informed approval decision. This is a key decision point requiring maximum user control.

6. **Plan Approval**: Use `AskUserQuestion` with options:
   - "Proceed with this plan"
   - "Modify the plan" (user describes changes)
   - "Add more tasks"

7. If user selects "Modify the plan" or "Add more tasks":
   - Wait for user input
   - Update the plan file accordingly
   - Re-present summary and ask again

8. **Finalize**: Add approval timestamp to progress log

**CRITICAL**: Do NOT proceed to Phase 6 until user explicitly approves the plan via `AskUserQuestion`.

**Output**: `claude-tmp/bug-fix-plan.md` file ready to guide implementation

---

## Phase 6: Fix Implementation

**Goal**: Implement the chosen fix following the approved plan

**Scope**: This phase works ONLY on `FIX-NNN` implementation tasks. Testing tasks (`TEST-NNN`) and review tasks (`REVIEW-NNN`) are handled in their respective phases.

**CRITICAL GATES** (verify before ANY implementation):
- [ ] Fix approach selected by user in Phase 4
- [ ] Plan approved via `AskUserQuestion` in Phase 5

If either gate is missing, STOP and complete the required phase first.

**Actions**:
1. **Verify both gates above are satisfied before writing any code**
2. **Read the plan file** (`claude-tmp/bug-fix-plan.md`) to get the task list
3. For each task in the "Fix Tasks" section:
   - Update state file with `currentTask: "FIX-NNN"`
   - Read all relevant files identified in previous phases
   - Implement the fix following:
     - Codebase conventions strictly
     - Minimal changes principle (don't refactor unrelated code)
     - Defensive coding where appropriate
   - **Mark task complete in plan file**: Change `- [ ]` to `- [x]`, add `Completed: [timestamp]` line
   - Add progress log entry: `| [timestamp] | FIX-NNN completed |`
4. Update todos as you progress
5. After implementation, briefly explain what was changed and why
6. **Update plan file after each task completion**

**Guidelines**:
- Follow existing code patterns exactly
- Don't add unrequested features
- Don't refactor unrelated code
- Keep implementations simple

**Output**: Implemented fix with updated plan file

---

## Phase 7: Testing

**Goal**: Ensure the fix is properly tested with user-approved strategy

**Scope**: This phase works ONLY on `TEST-NNN` testing tasks. Fix tasks (`FIX-NNN`) were completed in Phase 6. Review tasks (`REVIEW-NNN`) are handled in Phase 8.

**CRITICAL REQUIREMENT**: Every bug fix MUST include unit tests that would catch the bug. If a bug was found and fixed, add tests that:
- Would have **failed before the fix** (demonstrating the bug exists)
- **Pass after the fix** (proving the fix works)
- **Prevent regression** (ensure the bug cannot return undetected)

**Approach**: Use a `bug-test-analyzer` agent to propose test cases covering the bug scenario and edge cases, present analysis to user for approval, then write tests directly (not delegated) to preserve fix context. Use `bug-test-runner` agent to execute tests and report results.

**Actions**:
1. **Read the plan file** and review existing `TEST-NNN` tasks
2. Launch 1 `bug-test-analyzer` agent to analyze the fix:
   - **Identify the exact conditions that trigger the bug** - these become test cases
   - Propose test cases that specifically reproduce the bug scenario
   - Identify edge cases related to the bug
   - Note the total number of tests proposed
3. Present the test analysis to user:
   - Summarize the proposed test categories
   - **Highlight the bug reproduction test(s)** - the most important tests
   - List key test cases with their purposes
   - Highlight any mocking requirements or special setup needed
4. Use `AskUserQuestion` to get EXPLICIT confirmation:
   - Option 1: "Proceed with proposed testing strategy"
   - Option 2: "Modify testing scope" (user describes changes)
   - Option 3: "Skip testing phase" (only if testing is truly not feasible)
5. If user selects "Modify testing scope":
   - Wait for user to describe their modifications
   - Incorporate feedback into test plan
6. Write tests following the confirmed plan and project conventions:
   - **Bug reproduction test (REQUIRED)**: A test that exercises the exact scenario that caused the bug. This test would fail on the pre-fix code and passes with the fix.
   - Edge case tests: Cover related edge cases exposed by the bug
   - Regression prevention tests: Broader tests to guard against similar issues
   - **Mark each TEST-NNN task complete** in plan file as tests are written
7. **Execute tests using `bug-test-runner` agent**:
   - Pass the test files/patterns from the approved test plan
   - Agent runs tests and returns structured results
8. **Handle test results**:
   - If all tests pass, proceed to Phase 8
   - If tests fail:
     - Review the failure report from `bug-test-runner`
     - Fix the failing tests (adjust test code or implementation as needed)
     - Re-run `bug-test-runner` until all pass

**CRITICAL**: Do NOT write tests until user has made an explicit selection via `AskUserQuestion`.

**Why this approach**:
- Analyzer agent provides structured test proposals focused on regression prevention
- User approval ensures alignment with expectations before effort is spent
- Writing tests directly preserves local context from the fix implementation
- Test-runner agent provides structured, parseable test results

**Test Quality Standards**:
- **Bug reproduction test is mandatory** - at minimum, include one test that would have caught this specific bug
- Test names clearly describe what is being tested (e.g., `test_handles_null_input_that_caused_crash`)
- Each test focuses on a single behavior
- Tests are independent (no shared state)
- Follow existing project test patterns exactly

**Output**: User-approved test suite with passing tests, including at least one bug reproduction test

---

## Phase 8: Quality Review

**Goal**: Ensure fix quality and correctness through user-reviewed findings

**Scope**: This phase works ONLY on `REVIEW-NNN` quality tasks.

**Actions**:

### Step 1: Launch Review Agents
1. Launch 3 `bug-fix-reviewer` agents in parallel with different focuses:
   - Agent 1: Fix correctness (does it actually address root cause?)
   - Agent 2: Regression risk (could it break existing functionality?)
   - Agent 3: Edge cases and side effects
2. **Wait for ALL agents to complete** before proceeding

### Step 2: Consolidate Findings
1. Collect all findings from completed agents
2. Deduplicate overlapping issues (same file + line + similar description)
3. Organize by severity:
   - **Critical Issues** (Confidence 90-100): Must be fixed
   - **Important Issues** (Confidence 80-89): Should be addressed
   - **Suggestions** (Confidence < 80): Nice to have improvements

### Step 3: Present Findings to User
Display consolidated findings in a clear format:

```
## Review Findings Summary

### Critical Issues ({count})
1. [FILE:LINE] Description - {confidence}%
2. ...

### Important Issues ({count})
1. [FILE:LINE] Description - {confidence}%
2. ...

### Suggestions ({count})
1. [FILE:LINE] Description - {confidence}%
2. ...
```

### Step 4: User Selection
Use `AskUserQuestion` with `multiSelect: true` to let user choose which issues to address:
- List each issue as a selectable option
- Group by severity in the question
- Include "Skip all - proceed to summary" as an option

### Step 5: Apply Selected Fixes
1. Apply fixes ONLY for issues the user selected
2. Track which fixes were applied for the summary
3. **Mark REVIEW-NNN tasks complete** in plan file

### Step 6: Offer Re-review
If any fixes were applied, use `AskUserQuestion` to ask:
- "Run review again to verify fixes?"
- "Proceed to summary"

If user chooses re-review, return to Step 1 with a focused scope.

**Output**: Quality-verified fix with user-approved changes

---

## Phase 9: Summary

**Goal**: Document what was found and fixed, and clean up workflow state

**Actions**:
1. Mark all todos complete
2. Summarize:
   - **Root Cause**: What was actually causing the bug
   - **Fix Applied**: What changes were made and why
   - **Files Modified**: List of all changed files
   - **Tasks Completed**: Summary of FIX-NNN, TEST-NNN, REVIEW-NNN tasks
   - **Verification**: How to verify the fix works
   - **Prevention**: Suggestions to prevent similar bugs
   - **Follow-up**: Any remaining concerns or recommended next steps
3. **Delete workflow files** to mark workflow as complete:
   - `claude-tmp/bug-fix-state.json`
   - `claude-tmp/bug-fix-plan.md`

---
