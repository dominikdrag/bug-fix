---
description: Guided bug investigation and fixing workflow with codebase exploration and root cause analysis
argument-hint: Optional bug description or symptoms
---

# Bug Fix Workflow

You are helping a developer find and fix a bug or defect in their code. Follow a systematic approach: understand the bug thoroughly, explore the codebase, investigate potential causes, form hypotheses, and implement a fix with proper validation.

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
1. Launch all investigation agents **in parallel** (2-3 bug-investigator + 1 bug-historian):

   **Bug Investigator Agents (2-3x)** - analyze the code for issues:
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
1. Launch 2-3 bug-hypothesis agents in parallel with different focuses:
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

## Phase 5: Fix Implementation

**Goal**: Implement the chosen fix

**DO NOT START WITHOUT USER APPROVAL**

**Actions**:
1. Wait for explicit user approval on the fix approach
2. Read all relevant files identified in previous phases
3. Implement the fix following:
   - Codebase conventions strictly
   - Minimal changes principle (don't refactor unrelated code)
   - Defensive coding where appropriate
4. Update todos as you progress
5. After implementation, briefly explain what was changed and why

---

## Phase 6: Fix Review & Test Validation

**Goal**: Validate the fix is correct, doesn't introduce new issues, and passes tests

**Actions**:

### Part A: Parallel Validation (launch all 4 agents simultaneously)
1. Launch all validation agents **in parallel** in a single message:

   **Code Review Agents (3x bug-fix-reviewer)**:
   - Agent 1: Fix correctness (does it actually address root cause?)
   - Agent 2: Regression risk (could it break existing functionality?)
   - Agent 3: Edge cases and side effects

   **Test Validation Agent (1x bug-test-runner)**:
   - Discover the project's test framework
   - Find tests related to the modified files
   - Run related tests
   - Analyze results and identify coverage gaps
   - Suggest new tests if the fix lacks coverage

### Part B: Consolidate Results
2. Wait for all agents to complete
3. Consolidate findings:
   - Code review issues from all 3 reviewers
   - Test results (pass/fail/no tests)
   - Coverage assessment
   - Overall confidence level

### Part C: Decision
4. **Present consolidated findings to user**:
   - Code review issues (if any)
   - Test results (pass/fail/no tests)
   - Coverage gaps identified
   - Overall confidence level

5. **Ask what they want to do**:
   - Fix issues now (address code review issues or failing tests)
   - Add tests (if coverage gaps identified)
   - Fix later (note for future)
   - Proceed as-is (accept current state)

---

## Phase 7: Summary

**Goal**: Document what was found and fixed

**Actions**:
1. Mark all todos complete
2. Summarize:
   - **Root Cause**: What was actually causing the bug
   - **Fix Applied**: What changes were made and why
   - **Files Modified**: List of all changed files
   - **Verification**: How to verify the fix works
   - **Prevention**: Suggestions to prevent similar bugs
   - **Follow-up**: Any remaining concerns or recommended next steps

---
