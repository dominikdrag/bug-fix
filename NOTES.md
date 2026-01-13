# Implementation Notes

## Workflow State Persistence (Compaction Recovery)

The workflow uses a state file (`claude-tmp/bug-fix-state.json`) to persist progress across:
- Conversation compaction (automatic summarization when context gets long)
- Session interruptions

### Implementation Details

1. **PreCompact Hook**: Before compaction, writes current phase state to the file
2. **Workflow Start Check**: Command checks for existing state file to detect resumed workflows
3. **Phase Transitions**: State file updated at start/end of each phase
4. **Cleanup**: State file deleted on Phase 8 completion

### Auto-Approval Hook for State File Operations

A PreToolUse hook automatically approves Write/Edit/Bash operations on the state file without requiring user permission:

**Hook Configuration** (hooks/hooks.json):
- Matcher: `Write|Edit|Bash` (positioned first in PreToolUse array)
- Checks if operation targets `claude-tmp/bug-fix-state.json`
- Auto-approves matching operations
- Allows non-matching operations to pass through to subsequent hooks

**Rationale**: State file operations happen frequently (workflow start, phase transitions, compaction) and shouldn't interrupt user workflow with permission prompts.

### State File Schema

The state file includes full historical context via `phaseHistory`:

```json
{
  "active": true,
  "workflowType": "bug-fix",
  "bugDescription": "Login fails with 500 error when password contains special characters",
  "startedAt": "2026-01-13T10:30:00Z",
  "lastUpdatedAt": "2026-01-13T11:45:00Z",
  "currentPhase": 4,
  "currentTask": null,
  "phaseHistory": [
    {
      "phase": 1,
      "name": "Discovery",
      "status": "completed",
      "startedAt": "2026-01-13T10:30:00Z",
      "completedAt": "2026-01-13T10:35:00Z",
      "outputs": {
        "symptoms": ["500 error on login", "only with special chars"],
        "reproductionSteps": ["Enter password with @#$", "Submit form"],
        "expectedBehavior": "Login succeeds",
        "affectedArea": "Authentication module"
      }
    },
    {
      "phase": 2,
      "name": "Exploration",
      "status": "completed",
      "startedAt": "2026-01-13T10:35:00Z",
      "completedAt": "2026-01-13T10:50:00Z",
      "outputs": {
        "agentCount": 3,
        "keyFiles": ["src/auth/login.ts", "src/utils/sanitize.ts"],
        "executionPaths": ["POST /login → validateInput → hashPassword"],
        "areasOfConcern": ["Input sanitization in validateInput"]
      }
    },
    {
      "phase": 3,
      "name": "Investigation",
      "status": "completed",
      "startedAt": "2026-01-13T10:50:00Z",
      "completedAt": "2026-01-13T11:05:00Z",
      "outputs": {
        "agentCount": 4,
        "findings": [
          {"issue": "Unescaped regex in sanitizer", "confidence": 95, "file": "src/utils/sanitize.ts:42"}
        ],
        "historicalContext": "Introduced in commit abc123 during v2.0 migration"
      }
    },
    {
      "phase": 4,
      "name": "Hypothesis",
      "status": "in_progress",
      "startedAt": "2026-01-13T11:05:00Z",
      "completedAt": null,
      "outputs": {}
    }
  ],
  "decisions": {
    "fixApproach": null,
    "planApproved": null,
    "testStrategy": null
  },
  "summary": "Investigating hypothesis for regex escaping issue in sanitizer."
}
```

### Phase-Specific Outputs

| Phase | Outputs |
|-------|---------|
| 1 Discovery | `symptoms[]`, `reproductionSteps[]`, `expectedBehavior`, `affectedArea` |
| 2 Exploration | `agentCount`, `keyFiles[]`, `executionPaths[]`, `areasOfConcern[]` |
| 3 Investigation | `agentCount`, `findings[]`, `historicalContext` |
| 4 Hypothesis | `agentCount`, `hypotheses[]`, `selectedApproach`, `selectionRationale` |
| 5 Planning | `fixTaskCount`, `reviewTaskCount`, `planFile` |
| 6 Implementation | `tasksCompleted[]`, `tasksRemaining[]`, `filesModified[]` |
| 7 Testing | `testsWritten[]`, `testsPassing` |
| 8 Review | `issuesFound`, `issuesFixed[]`, `issuesSkipped[]` |
| 9 Summary | `filesModified[]`, `rootCause`, `prevention[]` |

### Recovery Behavior

When resuming:
1. User informed which phase is being resumed
2. Historical context from `phaseHistory` is displayed:
   - Key files and execution paths from exploration
   - Investigation findings with confidence scores
   - Historical context (culprit commit, original intent)
   - Selected hypothesis and fix approach
3. Workflow continues from current phase (no restart)

## Testing Phase (Phase 6)

The workflow now separates testing from review, following feature-dev's pattern:
- `bug-test-analyzer` agent proposes regression prevention tests
- User approval required via `AskUserQuestion` before writing tests
- Tests written directly (not delegated) to preserve fix context
- `bug-test-runner` agent executes tests and reports results

This ensures:
1. Comprehensive test coverage focused on preventing regression
2. User alignment with testing strategy before effort is spent
3. Better test quality by preserving implementation context
4. Structured, parseable test execution results

## Quality Review Phase (Phase 7)

The review phase now includes user-driven fix selection:
- 3 parallel `bug-fix-reviewer` agents with different focuses
- Findings consolidated by severity (Critical 90-100, Important 80-89)
- User selects which issues to fix via `AskUserQuestion` with multiSelect
- Optional re-review loop after fixes applied

This approach:
1. Enables parallel review for faster feedback
2. Organizes findings by actionability
3. Gives users control over which fixes to apply
4. Supports iterative improvement through re-review
