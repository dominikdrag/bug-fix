---
description: Investigate git history to find when and why code was written or changed
argument-hint: File path(s) or code area to investigate
---

# Git History Investigation

You are helping a developer understand the history of their code using git. This is useful for:
- Finding when a bug was introduced
- Understanding why code was written a certain way
- Identifying who to consult about specific code
- Tracing the evolution of a feature

## Input

Files or code areas to investigate: $ARGUMENTS

If no arguments provided, ask the user what files or code they want to investigate.

## Process

1. **Confirm scope**: Verify which files/code the user wants to investigate
2. **Launch bug-historian agent**:
   - Provide the file paths to investigate
   - Ask it to trace the git history and find key commits
   - Request analysis of original intent and any issues
3. **Present findings**:
   - Key commits that shaped the code
   - Timeline of changes
   - Original intent behind the code
   - Any notable patterns (frequent fixes, refactors, etc.)

## Example Prompts for the Agent

If investigating a specific file:
```
"Investigate the git history of [file] to understand when it was created,
major changes it has undergone, and the intent behind the current implementation"
```

If looking for when something changed:
```
"Use git log and git blame to find when [functionality/code pattern] was
introduced or last modified in [file/area]"
```

If tracking down a potential regression:
```
"Search git history to find commits that modified [affected code] in the
past [timeframe], looking for changes that could have introduced [symptom]"
```

## Output

Present a clear summary including:
- **Key Commits**: Important commits that shaped this code
- **Timeline**: When major changes happened
- **Authors**: Who worked on this code (for consultation)
- **Intent**: Why the code was written this way
- **Observations**: Any patterns worth noting (frequent changes, TODOs, etc.)
