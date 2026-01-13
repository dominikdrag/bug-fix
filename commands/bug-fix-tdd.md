---
description: TDD-style bug investigation and fixing workflow - write failing tests first, then implement fix
allowed-tools: Read, Write, Edit, Glob, Grep, LS, Bash, Agent, TodoWrite, AskUser
argument-hint: [--explorers=N] [--investigators=N] [--hypothesis=N] [--reviewers=N] <bug-description>
---

# TDD Bug Fix Workflow

You are helping a developer find and fix a bug using Test-Driven Development (TDD). The key difference from the regular bug-fix workflow is that tests are written FIRST to reproduce the bug, verified to FAIL, then the fix is implemented to make them PASS.

**TDD Mantra**: Red → Green → Refactor
- **Red**: Write a test that fails (proves the bug exists)
- **Green**: Write code to make the test pass (fixes the bug)
- **Refactor**: Clean up if needed (in Quality Review)

---

## Workflow State Management

This workflow uses a state file (`claude-tmp/bug-fix-tdd-state.json`) to persist progress across conversation compaction and session interruptions.

### On Workflow Start

**FIRST**, check if `claude-tmp/bug-fix-tdd-state.json` exists:
- **If file exists and `active: true`**: This is a RESUMED workflow
  - Read the state file to understand current progress
  - **If `claude-tmp/bug-fix-tdd-plan.md` exists**: Read the plan file
  - **If plan exists**: Parse it to determine current task progress
    - Count remaining unchecked tasks (lines matching `- [ ]`)
    - Display: "Current task: {currentTask.id} ({currentTask.substep}), {N} tasks remaining"
  - Inform the user: "Resuming TDD bug-fix workflow from Phase {currentPhase}"
  - Display historical context from `phaseHistory`:
    - For each completed phase, show phase name and key outputs
    - Phase 2: Show key files and execution paths discovered
    - Phase 3: Show investigation findings and historical context
    - Phase 4: Show selected hypothesis and approach
    - Phase 6: Show test design and approved test cases
  - Continue from the current phase (do NOT restart from Phase 1)
- **If file does not exist**: This is a NEW workflow
  - Create initial state file:
    ```json
    {
      "active": true,
      "workflowType": "bug-fix-tdd",
      "bugDescription": "[from user input]",
      "startedAt": "[current ISO timestamp]",
      "lastUpdatedAt": "[current ISO timestamp]",
      "currentPhase": 1,
      "currentTask": null,
      "phaseHistory": [],
      "decisions": {"fixApproach": null, "planApproved": null, "testDesignApproved": null, "testsVerifiedFailing": null},
      "summary": "Starting TDD bug-fix workflow"
    }
    ```
  - Proceed with Phase 1

### State File Format

```json
{
  "active": true,
  "workflowType": "bug-fix-tdd",
  "bugDescription": "...",
  "startedAt": "ISO timestamp",
  "lastUpdatedAt": "ISO timestamp",
  "currentPhase": 1,
  "currentTask": {
    "id": "TEST-001",
    "substep": "red|green|refactor",
    "attempts": 0,
    "errors": []
  },
  "phaseHistory": [
    {
      "phase": 1,
      "name": "Discovery",
      "status": "completed",
      "startedAt": "ISO timestamp",
      "completedAt": "ISO timestamp",
      "outputs": {
        "symptoms": ["symptom 1", "symptom 2"],
        "reproductionSteps": ["step 1", "step 2"],
        "expectedBehavior": "..."
      }
    }
  ],
  "decisions": {
    "fixApproach": null,
    "planApproved": null,
    "testDesignApproved": null,
    "testsVerifiedFailing": null
  },
  "summary": "Brief context for resumption"
}
```

### Phase-Specific Outputs

Each phase stores structured outputs in `phaseHistory[].outputs`:

| Phase | Name | Outputs |
|-------|------|---------|
| 1 | Discovery | `symptoms[]`, `reproductionSteps[]`, `expectedBehavior`, `affectedArea` |
| 2 | Exploration | `agentCount`, `keyFiles[]`, `executionPaths[]`, `areasOfConcern[]` |
| 3 | Investigation | `agentCount`, `findings[]`, `historicalContext` |
| 4 | Hypothesis | `agentCount`, `hypotheses[]`, `selectedApproach`, `selectionRationale` |
| 5 | Planning | `testTaskCount`, `fixTaskCount`, `reviewTaskCount`, `planFile` |
| 6 | Test Design | `testDesignApproved`, `testCases[]` |
| 7 | Red Phase | `testsWritten[]`, `testsVerifiedFailing` |
| 8 | Green Phase | `tasksCompleted[]`, `testsVerifiedPassing`, `filesModified[]` |
| 9 | Review | `issuesFound`, `issuesFixed[]`, `issuesSkipped[]` |
| 10 | Summary | `filesModified[]`, `rootCause`, `tddProcess` |

### Updating State

At the START of each phase, update the state file:
- Set `currentPhase` to the new phase number
- Set `lastUpdatedAt` to current ISO timestamp
- Add new entry to `phaseHistory[]` with `status: "in_progress"`, `startedAt`, and empty `outputs`
- Update `summary` with relevant context

At the END of each phase, update the state file:
- Update the phase's `phaseHistory` entry: set `status: "completed"`, `completedAt`, and populate `outputs`
- Store any decisions made in `decisions`

---

## Core Principles

- **Tests come first**: Write failing tests before writing fix code
- **Verify the red**: Confirm tests fail before implementing - this proves the bug exists
- **Minimal fix**: Write just enough code to make tests pass
- **Gather information first**: Understand the bug symptoms, reproduction steps, and expected behavior before exploring code
- **Understand before acting**: Read and comprehend existing code patterns and execution paths first
- **Read files identified by agents**: When launching agents, ask them to return lists of the most important files to read. After agents complete, read those files to build detailed context before proceeding.
- **Form hypotheses**: Don't jump to conclusions - form testable hypotheses based on evidence
- **Use TodoWrite**: Track all progress throughout

---

## Configuration

### Parse Arguments

Arguments: $ARGUMENTS

Parse optional flags to configure agent counts:
- `--explorers=N` - Number of `bug-explorer` agents (default: 3)
- `--investigators=N` - Number of `bug-investigator` agents (default: 3)
- `--hypothesis=N` - Number of `bug-hypothesis` agents (default: 3)
- `--reviewers=N` - Number of `bug-fix-reviewer` agents (default: 3)

Valid range: 1-10 for explorers, 1-5 for all others. If a value is out of range, use the closest value in range.

Remaining text after flags is the bug description.

### Display Configuration

At the start, confirm the configuration:
> **TDD Mode**: Tests will be written FIRST, verified to FAIL, then fix implemented
> Using agent counts: {explorers} explorers, {investigators} investigators, {hypothesis} hypothesis, {reviewers} reviewers

---

## Phase 1: Discovery

**Goal**: Understand the bug or defect thoroughly

Initial request: $ARGUMENTS

**Actions**:
1. Create todo list with all phases (note: this is a 10-phase TDD workflow)
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
1. Launch {explorers} bug-explorer agent(s) in parallel. Each agent should:
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
1. Launch all investigation agents **in parallel** ({investigators} bug-investigator + 1 bug-historian):

   **Bug Investigator Agents ({investigators}x)** - analyze the code for issues:
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
1. Launch {hypothesis} bug-hypothesis agent(s) in parallel with different focuses:
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

2. **Define Tasks**: Break down the chosen fix approach into discrete, actionable tasks:
   - Create tasks for each file to be modified
   - Establish task dependencies where needed
   - Assign task IDs:
     - `TEST-NNN` for test writing (Phase 7 - Red)
     - `FIX-NNN` for fix implementation (Phase 8 - Green)
     - `REVIEW-NNN` for quality review (Phase 9)

   **Phase Scope Rules** (document explicitly in plan):
   - **Phase 6 (Test Design)**: Analyze and design tests
   - **Phase 7 (Red Phase)**: Works ONLY on `TEST-NNN` tasks, verify tests FAIL
   - **Phase 8 (Green Phase)**: Works ONLY on `FIX-NNN` tasks, verify tests PASS
   - **Phase 9 (Review)**: Works ONLY on `REVIEW-NNN` tasks

3. **Define Acceptance Criteria**: Extract or derive acceptance criteria from the fix approach
   - Each criterion should be testable/verifiable
   - Assign IDs: `AC-NNN`
   - Must include: "Bug no longer reproduces under original conditions"

4. **Write Plan File**: Create `claude-tmp/bug-fix-tdd-plan.md` with this structure.

   **IMPORTANT**: Include FULL details from phases 1-4, not summaries. The plan file should be a complete reference that enables resumption without needing the state file.

   ```markdown
   # TDD Bug Fix Plan

   > **Status**: In Progress
   > **Current Task**: [none]
   > **TDD Phase**: Planning
   > **Created**: [ISO timestamp]
   > **Last Updated**: [ISO timestamp]

   ## Phase 1: Discovery

   ### Symptoms
   - [Full list of ALL symptoms identified]

   ### Reproduction Steps
   1. [Step 1 with full detail]
   2. [Step 2 with full detail]

   ### Expected vs Actual Behavior
   - **Expected**: [complete description]
   - **Actual**: [complete description]

   ### Affected Area
   [Full description of affected codebase area]

   ## Phase 2: Codebase Exploration

   ### Key Files
   - `path/to/file.ts` - [why it's relevant, what it contains]
   - `path/to/file2.ts` - [why it's relevant, what it contains]

   ### Execution Paths
   - [Entry point] → [intermediate steps] → [symptom location]
   - Include file:line references throughout

   ### Areas of Concern
   - [Area 1]: [full description]
   - [Area 2]: [full description]

   ### Agent Findings Summary
   [Full synthesis of what exploration agents discovered - preserve all relevant details]

   ## Phase 3: Investigation & History

   ### Investigation Findings
   | Finding | Confidence | File:Line | Evidence |
   |---------|-----------|-----------|----------|
   | [Finding 1] | 95% | `file.ts:42` | [Full evidence] |
   | [Finding 2] | 85% | `file.ts:78` | [Full evidence] |

   ### Historical Context
   - **Culprit Commit**: [hash] - [full description]
   - **Original Intent**: [complete explanation of why code was written this way]
   - **Previous Fix Attempts**: [if any, with details]

   ## Phase 4: Hypothesis Formation

   ### Hypotheses Presented
   1. **[Hypothesis A Name]**: [Full description with supporting evidence]
   2. **[Hypothesis B Name]**: [Full description with supporting evidence]

   ### Selected Fix Approach
   **Choice**: [Quick/Proper/Comprehensive Fix]

   **Rationale**: [Complete rationale for why this was chosen]

   **Key Decisions**:
   - [Decision 1 with full context]
   - [Decision 2 with full context]

   ## Phase Scope Rules (TDD)
   - **Phase 6 (Test Design)**: Analyze and design reproduction tests
   - **Phase 7 (Red Phase)**: Write `TEST-NNN` tasks, VERIFY TESTS FAIL
   - **Phase 8 (Green Phase)**: Implement `FIX-NNN` tasks, VERIFY TESTS PASS
   - **Phase 9 (Review)**: Work on `REVIEW-NNN` tasks

   ## Testing Tasks (Phase 7 - Red)
   - [ ] **TEST-001**: Bug reproduction test
     - Scenario: [exact scenario that triggers the bug]
     - Expected: Test should FAIL before fix
   - [ ] **TEST-002**: [edge case description]

   ## Fix Tasks (Phase 8 - Green)
   - [ ] **FIX-001**: [description]
     - Files: `path/to/file.ts:line`
     - Depends on: TEST-001 failing
   - [ ] **FIX-002**: [description]
     - Files: `path/to/file.ts`
     - Depends on: FIX-001

   ## Quality Tasks (Phase 9)
   - [ ] **REVIEW-001**: Run code reviewers
   - [ ] **REVIEW-002**: Apply selected fixes

   ## Acceptance Criteria
   - [ ] **AC-001**: Bug no longer reproduces under original conditions
   - [ ] **AC-002**: All reproduction tests pass
   - [ ] **AC-003**: [additional criterion]

   ## TDD Progress Log
   | Timestamp | Phase | Event |
   |-----------|-------|-------|
   | [ISO] | Planning | Planning phase completed |
   ```

5. **Present Full Plan**: Display the complete plan content to the user by reading and showing `claude-tmp/bug-fix-tdd-plan.md`

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

**Output**: `claude-tmp/bug-fix-tdd-plan.md` file ready to guide TDD implementation

---

## Phase 6: Test Design

**Goal**: Design the reproduction tests that will prove the bug exists

**Scope**: This phase designs the tests - actual writing happens in Phase 7 (Red Phase).

**CRITICAL GATES** (verify before test design):
- [ ] Fix approach selected by user in Phase 4
- [ ] Plan approved via `AskUserQuestion` in Phase 5

If either gate is missing, STOP and complete the required phase first.

**Actions**:
1. **Verify both gates above are satisfied before proceeding**
2. **Read the plan file** (`claude-tmp/bug-fix-tdd-plan.md`) to get the TEST-NNN task list
3. Launch 1 `bug-test-analyzer` agent to design reproduction tests:
   - **Focus on the EXACT conditions that trigger the bug** - these become test cases
   - Propose test cases that specifically reproduce the bug scenario
   - Identify edge cases related to the bug
   - Note mocking requirements and special setup

4. Present the test design to user:
   - **Highlight the bug reproduction test(s)** - the most important tests
   - List key test cases with their purposes
   - Explain what inputs/conditions will trigger the bug
   - Show expected behavior (what should fail)

5. Use `AskUserQuestion` to get EXPLICIT confirmation:
   - Option 1: "Proceed to Red Phase (write failing tests)"
   - Option 2: "Modify test design" (user describes changes)

6. If user selects "Modify test design":
   - Wait for user to describe their modifications
   - Incorporate feedback into test plan
   - Update plan file
   - Ask again

7. **Update state** with `testDesignApproved: true`

**Output**: Approved test design ready for Phase 7 (Red Phase)

---

## Phase 7: Red Phase (Write Failing Tests)

**Goal**: Write tests that FAIL - proving the bug exists

**TDD Principle**: A failing test is PROOF that the bug exists and can be detected.

**Scope**: This phase works ONLY on `TEST-NNN` testing tasks. The tests MUST FAIL at this point.

**CRITICAL GATES** (verify before writing tests):
- [ ] Test design approved in Phase 6

If gate is missing, STOP and complete Phase 6 first.

**Actions**:
1. **Verify test design is approved**
2. **Read the plan file** and review `TEST-NNN` tasks
3. Write tests following the approved test design and project conventions:
   - **Bug reproduction test (REQUIRED)**: Exercises the exact scenario that caused the bug
   - Edge case tests: Cover related edge cases exposed by the bug
   - Follow existing project test patterns exactly

4. For each TEST-NNN task:
   - Update state file with `currentTask: "TEST-NNN"`
   - Write the test
   - **Mark task complete in plan file**: Change `- [ ]` to `- [x]`, add `Completed: [timestamp]`
   - Add progress log entry: `| [timestamp] | Red | TEST-NNN written |`

5. **RUN THE TESTS** using `bug-test-runner` agent

6. **VERIFY TESTS FAIL** (Critical TDD Step):

   **If tests FAIL (expected)**:
   - This is correct! The failing tests prove the bug exists
   - Update state with `testsVerifiedFailing: true`
   - Add progress log entry: `| [timestamp] | Red | Tests verified FAILING - bug confirmed |`
   - Proceed to Phase 8 (Green Phase)

   **If tests PASS (unexpected)**:
   - **STOP IMMEDIATELY** - This means either:
     - The bug cannot be reproduced with this test
     - The bug was already fixed
     - The test doesn't capture the real bug condition
   - Use `AskUserQuestion` with options:
     - "Revisit test design" (go back to Phase 6)
     - "Investigate why bug can't be reproduced" (gather more info)
     - "Proceed anyway" (user confirms this is expected)
   - Do NOT proceed to Phase 8 until resolved

**Test Quality Standards**:
- **Bug reproduction test is mandatory** - at minimum, include one test that exercises the exact bug scenario
- Test names clearly describe what is being tested
- Each test focuses on a single behavior
- Tests are independent (no shared state)
- Follow existing project test patterns exactly

**Output**: Failing tests that prove the bug exists, ready for fix implementation

---

## Phase 8: Green Phase (Implement Fix)

**Goal**: Implement the fix to make the failing tests PASS

**TDD Principle**: Write just enough code to make the tests pass - no more.

**Scope**: This phase works ONLY on `FIX-NNN` implementation tasks.

**CRITICAL GATES** (verify before ANY implementation):
- [ ] Tests verified failing in Phase 7 (or user explicitly approved proceeding)

If gate is missing, STOP and complete Phase 7 first.

**Actions**:
1. **Verify tests are confirmed failing (or user approved proceeding)**
2. **Read the plan file** (`claude-tmp/bug-fix-tdd-plan.md`) to get the FIX-NNN task list
3. For each task in the "Fix Tasks" section:
   - Update state file with `currentTask: "FIX-NNN"`
   - Read all relevant files identified in previous phases
   - Implement the fix following:
     - **Minimal changes principle** - just enough to make tests pass
     - Codebase conventions strictly
     - Defensive coding where appropriate
   - **Mark task complete in plan file**: Change `- [ ]` to `- [x]`, add `Completed: [timestamp]`
   - Add progress log entry: `| [timestamp] | Green | FIX-NNN completed |`

4. **RUN THE TESTS** using `bug-test-runner` agent

5. **VERIFY TESTS PASS** (Critical TDD Step):

   **If tests PASS (expected)**:
   - Success! The fix works
   - Add progress log entry: `| [timestamp] | Green | Tests verified PASSING - fix confirmed |`
   - Proceed to Phase 9 (Quality Review)

   **If tests FAIL (unexpected)**:
   - The fix is incomplete or incorrect
   - Analyze the failure
   - Adjust the implementation
   - Re-run tests until they pass
   - Do NOT proceed until all tests pass

6. Update todos as you progress
7. After implementation, briefly explain what was changed and why
8. **Update plan file after each task completion**

**Guidelines**:
- Follow existing code patterns exactly
- Don't add unrequested features
- Don't refactor unrelated code
- Keep implementations minimal - just enough to pass tests

**Output**: Implemented fix with all tests passing

---

## Phase 9: Quality Review

**Goal**: Ensure fix quality and correctness through user-reviewed findings

**Scope**: This phase works ONLY on `REVIEW-NNN` quality tasks.

**Actions**:

### Step 1: Launch Review Agents
1. Launch {reviewers} `bug-fix-reviewer` agent(s) in parallel with different focuses:
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

### Step 6: Re-run Tests
After any changes:
1. Run tests again to ensure nothing broke
2. All tests must still pass

### Step 7: Offer Re-review
If any fixes were applied, use `AskUserQuestion` to ask:
- "Run review again to verify fixes?"
- "Proceed to summary"

If user chooses re-review, return to Step 1 with a focused scope.

**Output**: Quality-verified fix with user-approved changes and all tests passing

---

## Phase 10: Summary

**Goal**: Document what was found and fixed using TDD, and clean up workflow state

**Actions**:
1. Mark all todos complete
2. Summarize:
   - **Root Cause**: What was actually causing the bug
   - **TDD Process**:
     - Tests written to prove bug exists
     - Verification that tests failed initially (Red)
     - Fix implemented to make tests pass (Green)
   - **Fix Applied**: What changes were made and why
   - **Files Modified**: List of all changed files
   - **Tasks Completed**: Summary of TEST-NNN, FIX-NNN, REVIEW-NNN tasks
   - **Test Coverage**: Tests added for regression prevention
   - **Verification**: How to verify the fix works
   - **Prevention**: Suggestions to prevent similar bugs
   - **Follow-up**: Any remaining concerns or recommended next steps
3. **Delete workflow files** to mark workflow as complete:
   - `claude-tmp/bug-fix-tdd-state.json`
   - `claude-tmp/bug-fix-tdd-plan.md`

---
