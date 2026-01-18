---
description: TDD-style bug investigation and fixing workflow - write failing tests first, then implement fix
allowed-tools: Read, Write, Edit, Glob, Grep, LS, Bash, Agent, TodoWrite, AskUser
argument-hint: <bug-description>
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
| 1 | Discovery | `symptoms[]`, `reproductionSteps[]`, `expectedBehavior`, `affectedArea`, `clarificationRounds` |
| 2 | Exploration | `focusesSelected[]`, `agentCount`, `keyFiles[]`, `executionPaths[]`, `areasOfConcern[]` |
| 3 | Investigation | `focusesSelected[]`, `agentCount`, `findings[]`, `historicalContext` |
| 4 | Hypothesis | `focusesSelected[]`, `agentCount`, `hypotheses[]`, `selectedApproach`, `selectionRationale` |
| 5 | Planning | `testTaskCount`, `fixTaskCount`, `reviewTaskCount`, `planFile` |
| 6 | Test Design | `focusesSelected[]`, `testDesignApproved`, `testCases[]` |
| 7 | Red Phase | `testsWritten[]`, `testsVerifiedFailing` |
| 8 | Green Phase | `tasksCompleted[]`, `testsVerifiedPassing`, `filesModified[]` |
| 9 | Review | `focusesSelected[]`, `originalReviewTasks[]`, `newReviewTasks[]`, `issuesFound`, `issuesFixed[]`, `issuesSkipped[]` |
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

## Bug Description

Arguments: $ARGUMENTS

> **TDD Mode**: Tests will be written FIRST, verified to FAIL, then fix implemented

---

## Phase 1: Discovery

**Goal**: Understand the bug or defect thoroughly

Initial request: $ARGUMENTS

**Actions**:
1. Create todo list with all phases (note: this is a 10-phase TDD workflow)
2. **Initial Assessment**: Gather bug information from user:
   - What are the symptoms? (error messages, unexpected behavior, crashes)
   - How do you reproduce it? (steps, inputs, conditions)
   - When did it start? (recent change, always existed, intermittent)
   - What is the expected behavior vs actual behavior?
   - What area of the codebase is affected?
   - Any error logs, stack traces, or relevant output?

3. **Clarification Loop** (repeat until confident):
   a. Present focused questions for the current round (prioritize by impact on diagnosis)
   b. Wait for user answers
   c. Analyze answers:
      - Update understanding of the bug
      - Note any new ambiguities revealed by answers
      - Identify follow-up questions triggered by responses
   d. Re-assess understanding:
      - Are reproduction steps concrete and repeatable?
      - Is the symptom clearly defined?
      - Is the affected area bounded?
      - Are there remaining high-impact unknowns?
   e. **Confidence check**: Proceed if:
      - Reproduction steps are actionable
      - Symptom is clearly distinguishable from expected behavior
      - Affected area is sufficiently scoped
      - No critical information gaps remain
   f. If not confident, continue loop with follow-up questions

4. **Exit Criteria** - Proceed to Phase 2 when ALL of these are true:
   - Bug can be reproduced with documented steps
   - Symptom is clearly defined and observable
   - Expected vs actual behavior is unambiguous
   - Affected codebase area is identified
   - No answers contradict each other

5. Summarize the final understanding before proceeding

**Critical**: Get clear reproduction steps before proceeding. If the user doesn't have them, help them identify what triggers the bug through iterative dialogue. For TDD, clear reproduction steps are especially important because we need to write tests that reproduce the bug BEFORE fixing it.

---

## Phase 2: Codebase Exploration

**Goal**: Understand the code paths related to the bug

**Actions**:

### Step 1: Present Exploration Focus Options

Display available exploration focuses for user selection:

```
## Codebase Exploration

Select which focuses to explore. Each launches a parallel bug-explorer agent.

  Core Focuses (recommended for most bugs)
   1. Execution Flow       - Trace from trigger to symptom, identify all code involved
   2. Error Handling       - Map error paths, validation, exception handling
   3. State & Data Flow    - How data moves, state changes, side effects

  Specialized Focuses (use when relevant)
   4. Similar Code         - Find similar patterns that work correctly (comparison baseline)
   5. Dependencies         - External calls, libraries, shared components
   6. Configuration        - Settings, environment, feature flags that affect behavior

Enter selection (e.g., "1,2,3" or "1-3") [default: 1,2,3]:
```

Wait for user input before proceeding.

### Step 2: Parse Selection

Accept input formats:
- Comma-separated: `1,2,3`
- Ranges: `1-3`
- Mixed: `1-3,5`
- Empty/Enter: Use default `1,2,3`

Validate: numbers 1-6 only, deduplicate, sort.

### Step 3: Launch Selected Explorers

Launch one `bug-explorer` agent per selected focus, all in parallel.

**Agent Prompt Format**: Include the focus assignment in each agent's prompt:
> Your assigned exploration focus is: **[Focus Name]**
>
> [Include the FULL focus definition - see bug-fix.md Phase 2 for definitions]
>
> Bug context: [bug description and symptoms from Phase 1]
>
> Reproduction steps: [from Phase 1]
>
> Return a list of 5-10 key files to read.

**WAIT for ALL agent results** - Do NOT proceed until every launched agent has returned.

### Step 4: Synthesize Findings

1. Read all files identified by agents to build deep understanding
2. Present comprehensive summary of findings:
   - Execution flow with file:line references
   - Key components involved
   - Potential areas of concern identified
3. **Save the list of key files** for the historian agent in Phase 3

---

## Phase 3: Investigation & History

**Goal**: Identify potential root causes AND understand when/why the code was written

**CRITICAL**: This is where deep analysis happens. DO NOT SKIP.

**Actions**:

### Step 1: Present Investigation Focus Options

Display available investigation focuses for user selection:

```
## Bug Investigation

Select which investigation focuses to explore. Each launches a parallel bug-investigator agent.
Note: Bug historian runs automatically to analyze git history.

  Core Focuses (recommended for most bugs)
   1. Null/Undefined        - Missing null checks, optional chaining gaps, uninitialized variables
   2. Logic Errors          - Incorrect conditions, wrong operators, off-by-one errors
   3. Error Handling Gaps   - Unhandled exceptions, missing catch blocks, error swallowing

  Specialized Focuses (use when relevant)
   4. Race Conditions       - Async timing, state mutations during async ops, concurrent access
   5. State Management      - Stale closures, state inconsistency, missing synchronization
   6. Type & Data Issues    - Type coercion, validation gaps, format mismatches
   7. Resource Leaks        - Memory leaks, unclosed connections, missing cleanup
   8. API & Integration     - Incorrect API usage, assumption mismatches, version issues

Enter selection (e.g., "1,2,3" or "1-3") [default: 1,2,3]:
```

Wait for user input before proceeding.

### Step 2: Parse Selection

Accept input formats:
- Comma-separated: `1,2,3`
- Ranges: `1-3`
- Mixed: `1-3,5`
- Empty/Enter: Use default `1,2,3`

Validate: numbers 1-8 only, deduplicate, sort.

### Step 3: Launch Investigation Agents

Launch all investigation agents **in parallel**:

**Bug Investigator Agents** - one per selected focus:

**Agent Prompt Format**: Include the focus assignment in each agent's prompt:
> Your assigned investigation focus is: **[Focus Name]**
>
> [Include the FULL focus definition - see bug-fix.md Phase 3 for definitions]
>
> Bug context: [from Phase 1]
> Key files: [from Phase 2]
>
> Report only findings with confidence >= 70.

**Bug Historian Agent (1x)** - runs automatically:
> Investigate git history of [key files from Phase 2] to find when and why the suspicious code was written, and identify the commit that may have introduced the bug

**WAIT for ALL agent results** - Do NOT proceed until every launched agent has returned.

### Step 4: Consolidate Findings

1. Collect all findings from completed agents
2. Present findings with confidence scores:
   - Potential issues found with file:line references
   - Evidence supporting each finding
   - Prioritized by confidence level
   - **Historical context**: Culprit commit, original intent, previous fix attempts

---

## Phase 4: Hypothesis Formation

**Goal**: Form specific hypotheses about root cause and propose fix approaches

**Actions**:

### Step 1: Present Hypothesis Focus Options

Display available hypothesis perspectives for user selection:

```
## Hypothesis Formation

Select which hypothesis perspectives to explore. Each launches a parallel bug-hypothesis agent.

  Core Perspectives (recommended for most bugs)
   1. Primary Evidence       - Most likely cause based on strongest evidence
   2. Alternative Cause      - Second most likely cause, different root
   3. Systemic Analysis      - Whether this is symptomatic of broader issues

  Specialized Perspectives (use when relevant)
   4. Edge Case Focus        - Hypothesis centered on boundary conditions
   5. Timing & Order         - Hypothesis centered on async/timing issues
   6. Data Integrity         - Hypothesis centered on data corruption or format issues
   7. Integration Boundary   - Hypothesis centered on external system interactions
   8. Configuration/Env      - Hypothesis centered on environment-specific causes

Enter selection (e.g., "1,2,3" or "1-3") [default: 1,2,3]:
```

Wait for user input before proceeding.

### Step 2: Parse Selection

Accept input formats:
- Comma-separated: `1,2,3`
- Ranges: `1-3`
- Mixed: `1-3,5`
- Empty/Enter: Use default `1,2,3`

Validate: numbers 1-8 only, deduplicate, sort.

### Step 3: Launch Hypothesis Agents

Launch one `bug-hypothesis` agent per selected perspective, all in parallel.

**Agent Prompt Format**: Include the focus assignment in each agent's prompt:
> Your assigned hypothesis perspective is: **[Perspective Name]**
>
> [Include the FULL perspective definition - see bug-fix.md Phase 4 for definitions]
>
> Investigation findings: [from Phase 3]
> Historical context: [from Phase 3]
>
> Propose Quick Fix, Proper Fix, and Comprehensive Fix approaches.

**WAIT for ALL agent results** - Do NOT proceed until every launched agent has returned.

### Step 4: Synthesize and Present

1. Review all hypotheses and form your opinion on which is most likely correct
2. Present to user:
   - Primary hypothesis with supporting evidence
   - Alternative hypotheses if applicable
   - Fix approaches with trade-offs comparison
   - **Your recommendation with reasoning**
3. **Ask user which approach they prefer**

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
   - **Phase 9 (Review)**: Creates/refines `REVIEW-NNN` tasks from bug-fix-reviewer output, then executes them

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

   ## Review Tasks
   > Populated during Phase 9 (Quality Review) based on bug-fix-reviewer agent findings and user selection.

   (none yet)

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

**Scope**: This phase creates and executes `REVIEW-NNN` quality tasks.

**Actions**:

### Step 1: Present Review Focus Options

Display available review focuses for user selection:

```
## Quality Review

Select which review focuses to explore. Each launches a parallel bug-fix-reviewer agent.

  Core Focuses (recommended for most fixes)
   1. Fix Correctness        - Does fix address root cause? Could bug still occur?
   2. Regression Risk        - Could fix break existing functionality?
   3. Edge Cases             - Are boundary conditions handled? Missing cases?

  Specialized Focuses (use when relevant)
   4. Side Effect Analysis   - Unintended consequences, shared state impacts
   5. Convention Compliance  - Project guidelines, code style, patterns
   6. Completeness           - All affected paths fixed? Cleanup handled?

Enter selection (e.g., "1,2,3" or "1-3") [default: 1,2,3]:
```

Wait for user input before proceeding.

### Step 2: Parse Selection

Accept input formats:
- Comma-separated: `1,2,3`
- Ranges: `1-3`
- Mixed: `1-3,5`
- Empty/Enter: Use default `1,2,3`

Validate: numbers 1-6 only, deduplicate, sort.

### Step 3: Launch Review Agents

Launch one `bug-fix-reviewer` agent per selected focus, all in parallel.

**Agent Prompt Format**: Include the focus assignment in each agent's prompt:
> Your assigned review focus is: **[Focus Name]**
>
> [Include the FULL focus definition from bug-fix.md Phase 8]
>
> Files modified: [from Phase 6]
> Tests written: [from Phase 7-8]
>
> Report only issues with confidence >= 80.

**WAIT for ALL agent results** - Do NOT proceed until every launched agent has returned.

### Step 4: Consolidate Findings

1. Collect all findings from completed agents
2. Deduplicate overlapping issues (same file + line + similar description)
3. Organize by severity:
   - **Critical Issues** (Confidence 90-100): Must be fixed
   - **Important Issues** (Confidence 80-89): Should be addressed
   - **Suggestions** (Confidence < 80): Nice to have improvements

### Step 5: Reconcile with Plan

1. Read `claude-tmp/bug-fix-tdd-plan.md` and extract existing `REVIEW-NNN` tasks (if any from placeholder)
2. Map high-confidence issues (>=80%) to actionable tasks:
   - Match issues to existing REVIEW-NNN tasks where applicable
   - Identify issues that need new REVIEW-NNN tasks
   - Note any original REVIEW-NNN tasks that are no longer relevant
3. Prepare reconciliation summary for user presentation

### Step 6: Present Reconciled Findings to User

Display reconciled findings in a clear format:

```
## Review Findings Summary

### Original REVIEW Tasks (from plan)
- REVIEW-001: [description] - [keep/modify/complete]
- REVIEW-002: [description] - [keep/modify/complete]

### Proposed New REVIEW Tasks (from reviewer findings)
- REVIEW-003: [specific issue from findings] - [file:line]
- REVIEW-004: [specific issue from findings] - [file:line]

### Critical Issues ({count})
1. [FILE:LINE] Description - {confidence}%
   - Maps to: REVIEW-00N (new/existing)
2. ...

### Important Issues ({count})
1. [FILE:LINE] Description - {confidence}%
   - Maps to: REVIEW-00N (new/existing)
2. ...

### Suggestions ({count})
1. [FILE:LINE] Description - {confidence}%
2. ...
```

### Step 7: User Selection

Use `AskUserQuestion` with `multiSelect: true` to let user choose which issues to address:
- List each issue as a selectable option
- Group by severity in the question
- Include "Skip all - proceed to summary" as an option

### Step 8: Update Plan and Apply Selected Fixes

1. Update `claude-tmp/bug-fix-tdd-plan.md`:
   - Add new REVIEW-NNN tasks based on user selection
   - Mark skipped original REVIEW-NNN tasks as complete (if user chose to skip)
   - Each selected issue becomes a trackable task with sequential ID
2. Add progress log entry: `| [timestamp] | Review tasks refined based on reviewer output |`
3. Apply fixes ONLY for issues the user selected
4. For each fix applied:
   - Mark corresponding REVIEW-NNN task complete in plan file
   - Track which fixes were applied for the summary

### Step 9: Re-run Tests

After any changes:
1. Run tests again to ensure nothing broke
2. All tests must still pass (both bug reproduction tests and regression tests)

### Step 10: Offer Re-review

If any fixes were applied, use `AskUserQuestion` to ask:
- "Run review again to verify fixes?"
- "Proceed to summary"

If user chooses re-review, return to Step 3 with a focused scope.

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
