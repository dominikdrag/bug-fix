---
description: Guided bug investigation and fixing workflow with codebase exploration and root cause analysis
allowed-tools: Read, Write, Edit, Glob, Grep, LS, Bash, Agent, TodoWrite, AskUser
argument-hint: <bug-description>
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
  - **If `claude-tmp/bug-fix-plan.md` exists**: Read the plan file
  - **If plan exists**: Parse it to determine current task progress
    - Count remaining unchecked tasks (lines matching `- [ ]`)
    - Display: "Current task: {currentTask}, {N} tasks remaining"
  - Inform the user: "Resuming bug-fix workflow from Phase {currentPhase}"
  - Display historical context from `phaseHistory`:
    - For each completed phase, show phase name and key outputs
    - Phase 2: Show key files and execution paths discovered
    - Phase 3: Show investigation findings and historical context
    - Phase 4: Show selected hypothesis and approach
  - Continue from the current phase (do NOT restart from Phase 1)
- **If file does not exist**: This is a NEW workflow
  - Create initial state file:
    ```json
    {
      "active": true,
      "workflowType": "bug-fix",
      "bugDescription": "[from user input]",
      "startedAt": "[current ISO timestamp]",
      "lastUpdatedAt": "[current ISO timestamp]",
      "currentPhase": 1,
      "currentTask": null,
      "phaseHistory": [],
      "decisions": {"fixApproach": null, "planApproved": null, "testStrategy": null},
      "summary": "Starting bug-fix workflow"
    }
    ```
  - Proceed with Phase 1

### State File Format

```json
{
  "active": true,
  "workflowType": "bug-fix",
  "bugDescription": "...",
  "startedAt": "ISO timestamp",
  "lastUpdatedAt": "ISO timestamp",
  "currentPhase": 1,
  "currentTask": null,
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
        "expectedBehavior": "...",
        "affectedArea": "..."
      }
    }
  ],
  "decisions": {
    "fixApproach": null,
    "planApproved": null,
    "testStrategy": null
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
| 5 | Planning | `fixTaskCount`, `reviewTaskCount`, `planFile` |
| 6 | Implementation | `tasksCompleted[]`, `tasksRemaining[]`, `filesModified[]` |
| 7 | Testing | `focusesSelected[]`, `testsWritten[]`, `testsPassing` |
| 8 | Review | `focusesSelected[]`, `originalReviewTasks[]`, `newReviewTasks[]`, `issuesFound`, `issuesFixed[]`, `issuesSkipped[]` |
| 9 | Summary | `filesModified[]`, `rootCause`, `prevention[]` |

### Updating State

At the START of each phase, update the state file:
- Set `currentPhase` to the new phase number
- Set `lastUpdatedAt` to current ISO timestamp
- Add new entry to `phaseHistory[]` with `status: "in_progress"`, `startedAt`, and empty `outputs`
- Update `summary` with relevant context

At the END of each phase, update the state file:
- Update the phase's `phaseHistory` entry: set `status: "completed"`, `completedAt`, and populate `outputs`
- Store any decisions made (fix approach selection, plan approval, test strategy approval) in `decisions`

---

## Core Principles

- **Gather information first**: Understand the bug symptoms, reproduction steps, and expected behavior before exploring code
- **Understand before acting**: Read and comprehend existing code patterns and execution paths first
- **Read files identified by agents**: When launching agents, ask them to return lists of the most important files to read. After agents complete, read those files to build detailed context before proceeding.
- **Form hypotheses**: Don't jump to conclusions - form testable hypotheses based on evidence
- **Validate fixes thoroughly**: Ensure fixes address root cause, not just symptoms
- **Use TodoWrite**: Track all progress throughout

---

## Bug Description

Arguments: $ARGUMENTS

---

## Phase 1: Discovery

**Goal**: Understand the bug or defect thoroughly

Initial request: $ARGUMENTS

**Actions**:
1. Create todo list with all phases
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

**Critical**: Get clear reproduction steps before proceeding. If the user doesn't have them, help them identify what triggers the bug through iterative dialogue.

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
> [Include the FULL focus definition from the reference below]
>
> Bug context: [bug description and symptoms from Phase 1]
>
> Reproduction steps: [from Phase 1]
>
> Return a list of 5-10 key files to read.

**Focus Definitions** (inject the selected one into each agent prompt):

1. **Execution Flow**
   - Focus: Trace the complete path from trigger to symptom
   - When to use: Always - establishes baseline understanding
   - Start from the trigger point (user action, API call, event)
   - Follow the execution path through all layers
   - Identify branch points and conditional logic
   - Document the exact location where symptom manifests
   - Note any async operations, callbacks, or event handlers

2. **Error Handling**
   - Focus: Map error paths and exception handling in affected code
   - When to use: Bugs involving exceptions, silent failures, or unexpected error states
   - Identify try/catch blocks and error boundaries
   - Document what errors are caught vs propagated
   - Find missing error handling (null checks, validation)
   - Note error recovery and fallback logic
   - Check for swallowed exceptions or generic error handlers

3. **State & Data Flow**
   - Focus: Understand how data moves and state changes
   - When to use: Bugs involving incorrect data, state corruption, or timing issues
   - Trace data from input to where bug manifests
   - Identify state mutations and their triggers
   - Map shared state and potential race conditions
   - Document caching, memoization, or derived state
   - Note any side effects during data transformation

4. **Similar Code**
   - Focus: Find similar patterns that work correctly as comparison baseline
   - When to use: When the bug affects one case but similar cases work
   - Search for analogous features or operations
   - Compare implementation differences
   - Identify what the working code does differently
   - Note any guards or checks present in working code but missing in buggy code

5. **Dependencies**
   - Focus: Map external calls and shared components
   - When to use: Bugs that may involve library behavior, API changes, or shared utilities
   - Identify all external dependencies in the affected path
   - Check version constraints and recent updates
   - Document shared utilities and their contracts
   - Note any assumptions about external behavior

6. **Configuration**
   - Focus: Settings and environment factors affecting behavior
   - When to use: Bugs that appear in some environments but not others
   - Identify all configuration affecting the buggy code
   - Document feature flags and their states
   - Check environment-specific settings
   - Note any default values and override mechanisms

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
> [Include the FULL focus definition from the reference below]
>
> Bug context: [from Phase 1]
> Key files: [from Phase 2]
>
> Report only findings with confidence >= 70.

**Focus Definitions** (inject the selected one into each agent prompt):

1. **Null/Undefined**
   - Focus: Find missing null/undefined checks that could cause failures
   - When to use: Always - one of the most common bug patterns
   - Missing null checks before property access
   - Optional chaining gaps
   - Undefined array/object access
   - Uninitialized variables
   - Null propagation through function calls

2. **Logic Errors**
   - Focus: Identify incorrect logic that produces wrong behavior
   - When to use: Always - fundamental investigation category
   - Incorrect conditional expressions
   - Wrong comparison operators (== vs ===, > vs >=)
   - Inverted boolean logic
   - Missing or incorrect boundary conditions
   - Off-by-one errors in loops/indices

3. **Error Handling Gaps**
   - Focus: Find missing or incomplete error handling
   - When to use: When symptoms include silent failures or unexpected crashes
   - Unhandled promise rejections
   - Missing catch blocks for throwing code
   - Error swallowing without logging
   - Incorrect error propagation
   - Missing finally cleanup

4. **Race Conditions**
   - Focus: Identify timing-related issues in async code
   - When to use: When bug is intermittent or involves async operations
   - Async operations without proper awaiting
   - State mutations during async operations
   - Event handler timing issues
   - Concurrent access to shared state
   - Missing synchronization primitives

5. **State Management**
   - Focus: Find state-related issues causing inconsistent behavior
   - When to use: When behavior differs based on state or previous actions
   - State not updated when expected
   - Stale closures capturing old values
   - Mutations of supposedly immutable data
   - State inconsistency across components
   - Missing state synchronization

6. **Type & Data Issues**
   - Focus: Identify type and data format problems
   - When to use: When bug involves data transformation or type handling
   - Type coercion problems
   - Incorrect type assumptions
   - Missing validation
   - Data format mismatches
   - Encoding/decoding errors

7. **Resource Leaks**
   - Focus: Find resource management issues
   - When to use: When bug involves memory growth, performance degradation, or resource exhaustion
   - Memory leaks (unclosed listeners, timers)
   - Connection leaks
   - File handle leaks
   - Unbounded growth (caches, arrays)
   - Missing cleanup on unmount/exit

8. **API & Integration**
   - Focus: Identify issues with external interfaces
   - When to use: When bug involves external dependencies or API usage
   - Incorrect API usage
   - Missing error handling for API calls
   - Assumption mismatch with dependencies
   - Version incompatibilities
   - Configuration errors

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
> [Include the FULL perspective definition from the reference below]
>
> Investigation findings: [from Phase 3]
> Historical context: [from Phase 3]
>
> Propose Quick Fix, Proper Fix, and Comprehensive Fix approaches.

**Perspective Definitions** (inject the selected one into each agent prompt):

1. **Primary Evidence**
   - Focus: Form hypothesis based on the strongest available evidence
   - When to use: Always - establishes the most likely root cause
   - Synthesize all high-confidence findings from investigation
   - Form the most supported hypothesis
   - Provide complete causal chain from root cause to symptom
   - Include Quick/Proper/Comprehensive fix approaches

2. **Alternative Cause**
   - Focus: Consider the second most likely root cause
   - When to use: When there are multiple plausible explanations
   - Look for evidence that could support a different root cause
   - Consider what else could explain the symptoms
   - Identify what would distinguish this from the primary hypothesis
   - Provide fix approaches for this alternative

3. **Systemic Analysis**
   - Focus: Consider whether this bug indicates deeper issues
   - When to use: For recurring bugs or patterns that appear in multiple places
   - Look for similar patterns elsewhere in the codebase
   - Consider whether the bug is symptomatic of architectural issues
   - Identify related fragile code that might have the same problem
   - Propose fixes that address the systemic issue

4. **Edge Case Focus**
   - Focus: Form hypothesis centered on boundary conditions
   - When to use: When bug only appears with specific inputs or at boundaries
   - Focus on boundary conditions and edge cases
   - Consider off-by-one scenarios
   - Analyze empty/null/max value handling
   - Propose fixes specifically addressing edge cases

5. **Timing & Order**
   - Focus: Form hypothesis centered on timing-related issues
   - When to use: When bug is intermittent or involves async operations
   - Focus on async operation ordering
   - Consider race condition scenarios
   - Analyze event timing dependencies
   - Propose fixes with synchronization or ordering guarantees

6. **Data Integrity**
   - Focus: Form hypothesis centered on data corruption or format issues
   - When to use: When bug involves incorrect or corrupted data
   - Focus on data transformation steps
   - Consider type coercion and format issues
   - Analyze validation and sanitization gaps
   - Propose fixes with data validation and type safety

7. **Integration Boundary**
   - Focus: Form hypothesis centered on external system interactions
   - When to use: When bug involves API calls or external services
   - Focus on integration points and contracts
   - Consider API response handling and error cases
   - Analyze timeout and retry scenarios
   - Propose fixes with better error handling and fallbacks

8. **Configuration/Environment**
   - Focus: Form hypothesis centered on environment-specific causes
   - When to use: When bug only occurs in specific environments or configurations
   - Focus on configuration differences
   - Consider environment-specific behavior
   - Analyze feature flags and conditional code
   - Propose fixes that work across environments

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

2. **Define Fix Tasks**: Break down the chosen fix approach into discrete, actionable tasks:
   - Create tasks for each file to be modified
   - Establish task dependencies where needed
   - Assign task IDs: `FIX-NNN` for fix implementation, `REVIEW-NNN` for quality
   - **Note**: `TEST-NNN` tasks are created dynamically in Phase 7 by the test-analyzer

   **Phase Scope Rules** (document explicitly in plan):
   - **Phase 6 (Implementation)**: Works ONLY on `FIX-NNN` tasks
   - **Phase 7 (Testing)**: Creates `TEST-NNN` tasks from test-analyzer output, then executes them
   - **Phase 8 (Review)**: Works ONLY on `REVIEW-NNN` tasks

3. **Define Acceptance Criteria**: Extract or derive acceptance criteria from the fix approach
   - Each criterion should be testable/verifiable
   - Assign IDs: `AC-NNN`
   - Must include: "Bug no longer reproduces under original conditions"

4. **Write Plan File**: Create `claude-tmp/bug-fix-plan.md` with this structure.

   **IMPORTANT**: Include FULL details from phases 1-4, not summaries. The plan file should be a complete reference that enables resumption without needing the state file.

   ```markdown
   # Bug Fix Plan

   > **Status**: In Progress
   > **Current Task**: [none]
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

   ## Phase Scope Rules
   - **Phase 6 (Implementation)**: Works ONLY on `FIX-NNN` tasks
   - **Phase 7 (Testing)**: Creates `TEST-NNN` tasks from test-analyzer output, then executes them (tasks are created dynamically, not during planning)
   - **Phase 8 (Review)**: Creates/refines `REVIEW-NNN` tasks from bug-fix-reviewer output, then executes them (tasks are created dynamically based on reviewer findings)

   ## Fix Tasks
   - [ ] **FIX-001**: [description]
     - Files: `path/to/file.ts:line`
     - Depends on: none
   - [ ] **FIX-002**: [description]
     - Files: `path/to/file.ts`
     - Depends on: FIX-001

   ## Test Tasks
   > Populated during Phase 7 (Testing) based on bug-test-analyzer agent output and user selection.

   (none yet)

   ## Review Tasks
   > Populated during Phase 8 (Quality Review) based on bug-fix-reviewer agent findings and user selection.

   (none yet)

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

**Scope**: This phase creates and executes `TEST-NNN` testing tasks. Fix tasks (`FIX-NNN`) were completed in Phase 6. Review tasks (`REVIEW-NNN`) are handled in Phase 8.

**CRITICAL REQUIREMENT**: Every bug fix MUST include unit tests that would catch the bug. If a bug was found and fixed, add tests that:
- Would have **failed before the fix** (demonstrating the bug exists)
- **Pass after the fix** (proving the fix works)
- **Prevent regression** (ensure the bug cannot return undetected)

**Approach**: Launch bug-test-analyzer agent to propose test cases based on the implemented fix. Present proposals to user for approval. **Once user approves the testing strategy, add `TEST-NNN` tasks to `claude-tmp/bug-fix-plan.md`** (this is when testing tasks are created - NOT during Phase 5 planning). Then write tests directly (not delegated) to preserve fix context. Use `bug-test-runner` agent to execute tests.

**Actions**:

### Step 1: Launch Test Analysis
1. Launch 1 `bug-test-analyzer` agent to analyze the implemented fix:
   - **Identify the exact conditions that trigger the bug** - these become test cases
   - Propose test cases that specifically reproduce the bug scenario
   - Identify edge cases related to the bug
   - Note mocking requirements and special setup needed
2. **Wait for agent to complete**

### Step 2: Present to User
1. Present the FULL output from test-analyzer agent - do NOT summarize
2. Present the proposed testing strategy:
   - Test cases organized by type (bug reproduction, edge cases, regression prevention)
   - **Highlight the bug reproduction test(s)** - the most important tests
   - Mocking requirements identified
   - Coverage expectations

**IMPORTANT**: Present the FULL output from test-analyzer agent to the user - do NOT summarize or condense the test proposals. The user needs complete visibility into each proposed test case, its rationale, edge cases identified, and mocking requirements to make an informed decision about the testing strategy.

### Step 3: User Approval
Use `AskUserQuestion` to get EXPLICIT confirmation:
- Option 1: "Proceed with proposed testing strategy"
- Option 2: "Modify testing scope" (user describes changes)
- Option 3: "Skip testing phase" (only if testing is truly not feasible)

### Step 4: Create TEST Tasks in Plan File

**IMPORTANT**: This is when `TEST-NNN` tasks are created and added to the plan file - NOT during Phase 5 planning. This ensures test proposals are based on actual implementation context.

If user approves the testing strategy:
1. Update `claude-tmp/bug-fix-plan.md`:
   - Add "## Testing Tasks" section after Fix Tasks (and before Quality Tasks)
   - Create `TEST-NNN` tasks based on approved test proposals (starting from TEST-001)
   - Each task should specify: test file path, what behavior is tested, edge cases covered
2. Add progress log entry: `| [timestamp] | Testing tasks created from test-analyzer output |`

### Step 5: Execute Testing Tasks
For each TEST-NNN task in the updated plan:
1. Update state file with `currentTask: "TEST-NNN"`
2. Write tests following the task description and project conventions:
   - **Bug reproduction test (REQUIRED)**: A test that exercises the exact scenario that caused the bug
   - Edge case tests: Cover related edge cases exposed by the bug
   - Regression prevention tests: Broader tests to guard against similar issues
3. Mark task complete: Change `- [ ]` to `- [x]`, add `Completed: [timestamp]`
4. Add progress log entry: `| [timestamp] | TEST-NNN completed |`

### Step 6: Run Tests
1. Launch `bug-test-runner` agent to execute all new tests
2. If tests fail:
   - Review the failure report
   - Fix the failing tests (adjust test code or implementation as needed)
   - Re-run until all pass
3. If all tests pass, proceed to Phase 8

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
> [Include the FULL focus definition from the reference below]
>
> Files modified: [from Phase 6]
> Tests written: [from Phase 7]
>
> Report only issues with confidence >= 80.

**Focus Definitions** (inject the selected one into each agent prompt):

1. **Fix Correctness**
   - Focus: Verify the fix actually addresses the root cause
   - When to use: Always - fundamental review responsibility
   - Does the fix address the identified root cause or just mask symptoms?
   - Will it work for all cases, not just the reported scenario?
   - Are there conditions where the bug could still occur?
   - Is the fix logically sound?

2. **Regression Risk**
   - Focus: Assess risk of breaking existing functionality
   - When to use: Always - prevents introducing new bugs
   - Could this fix break existing functionality?
   - Are there callers that depend on the previous behavior?
   - Have edge cases been considered?
   - Is backward compatibility maintained where needed?

3. **Edge Cases**
   - Focus: Verify handling of boundary conditions
   - When to use: Always - edge cases are common bug sources
   - Are boundary conditions handled?
   - What about null/undefined/empty inputs?
   - Are error cases properly handled?
   - What about concurrent access (if applicable)?

4. **Side Effect Analysis**
   - Focus: Identify unintended consequences of the fix
   - When to use: When fix modifies shared code or state
   - What other code paths are affected by this change?
   - Are there unintended consequences?
   - Does the fix modify shared state or data?
   - Could it affect performance?

5. **Convention Compliance**
   - Focus: Verify adherence to project guidelines
   - When to use: When fix touches multiple areas or involves new patterns
   - Does the fix follow project guidelines (CLAUDE.md)?
   - Are naming conventions followed?
   - Is error handling consistent with the codebase?
   - Is the code style consistent?

6. **Completeness**
   - Focus: Verify all aspects of the fix are complete
   - When to use: For comprehensive fixes or multi-file changes
   - Are all affected code paths fixed?
   - Is cleanup/teardown handled properly?
   - Are related issues addressed or noted?
   - Is documentation updated if needed?

**WAIT for ALL agent results** - Do NOT proceed until every launched agent has returned.

### Step 4: Consolidate Findings

1. Collect all findings from completed agents
2. Deduplicate overlapping issues (same file + line + similar description)
3. Organize by severity:
   - **Critical Issues** (Confidence 90-100): Must be fixed
   - **Important Issues** (Confidence 80-89): Should be addressed
   - **Suggestions** (Confidence < 80): Nice to have improvements

### Step 5: Reconcile with Plan

1. Read `claude-tmp/bug-fix-plan.md` and extract existing `REVIEW-NNN` tasks (if any from placeholder)
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

1. Update `claude-tmp/bug-fix-plan.md`:
   - Add new REVIEW-NNN tasks based on user selection
   - Mark skipped original REVIEW-NNN tasks as complete (if user chose to skip)
   - Each selected issue becomes a trackable task with sequential ID
2. Add progress log entry: `| [timestamp] | Review tasks refined based on reviewer output |`
3. Apply fixes ONLY for issues the user selected
4. For each fix applied:
   - Mark corresponding REVIEW-NNN task complete in plan file
   - Track which fixes were applied for the summary

### Step 9: Offer Re-review

If any fixes were applied, use `AskUserQuestion` to ask:
- "Run review again to verify fixes?"
- "Proceed to summary"

If user chooses re-review, return to Step 3 with a focused scope.

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
