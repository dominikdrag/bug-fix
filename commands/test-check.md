---
description: Run tests related to recent code changes and analyze results
allowed-tools: Read, Glob, Grep, LS, Bash, Agent, TodoWrite
argument-hint: [file-path...]
---

# Test Check

Run tests to validate code changes, analyze results, and identify coverage gaps.

## Usage

This command can be used in several ways:

1. **Test recent changes**: Run without arguments to test files changed since last commit
2. **Test specific files**: Provide file paths to test specific code
3. **Test a pattern**: Provide a glob pattern to test matching files

Initial request: $ARGUMENTS

## Workflow

### Step 1: Identify What to Test

**If no arguments provided:**
- Check `git status` and `git diff` to find recently modified files
- Identify the code files that have changed

**If arguments provided:**
- Use the specified files or patterns
- Validate the files exist

### Step 2: Launch Test Runner

Launch the bug-test-runner agent to:
- Detect the project's test framework
- Find tests related to the target files
- Run the related tests
- Analyze results
- Identify coverage gaps

**Example agent prompt:**
"Find and run tests related to [files]. Report results, analyze any failures, and identify if the changed code has adequate test coverage."

### Step 3: Present Results

Present a summary including:
- **Test Framework**: What was detected
- **Tests Run**: How many tests were executed
- **Results**: Pass/fail status
- **Failures**: Details on any failing tests
- **Coverage**: Whether the changes are covered by tests
- **Suggestions**: New tests that should be added

### Step 4: Offer Next Steps

Based on results, offer appropriate actions:

**If tests pass:**
- Confirm changes are safe
- Note any coverage gaps

**If tests fail:**
- Analyze if failures are caused by recent changes or pre-existing
- Offer to help fix failing tests

**If no tests found:**
- Note the risk of untested code
- Offer to help write tests

## Examples

```bash
# Test recent changes
/test-check

# Test specific file
/test-check src/services/auth.ts

# Test multiple files
/test-check src/api/users.ts src/api/auth.ts

# Test a directory
/test-check src/components/
```
