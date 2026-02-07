- Update `CLAUDE.md` when necessary
    - global at `~/.claude/CLAUDE.md`
    - local at `.claude/CLAUDE.md`

- Read @remote.md, if running from the Web interface (https://claude.ai/code/)
    - ignore if running from Claude CLI

- Stay in "Plan Mode". Always. After any edits, go back to "Plan Mode".

<!--
- As soon as `claude` starts, it should permanently change the terminal title
  to `"[CC] <current directory>"`.
-->

# Persistent Plans

- Each session has an associated plan `.claude/plan/<name>.md`
    - On startup, ask for `<name>`
- The plan must be continuously updated during sessions
- The plan should keep pending commands/actions that were not applied yet

# Plan Mode Workflow

- Always remain in "Plan Mode".
- Never edit files without explicit user authorization.
- Make suggestions/corrections/alerts before edits.
- After completing authorized edits
    - print a summary of places changed:
        - "<file>:<line> | place | description"
        - place can be a function, data structure, etc
        - use 1 line fore each change
    - return to plan mode automatically
        - show a message "I'm back to plan mode..."

# Language

- **English**: All answers, comments, and any other kind of output should be in
    English.
- I may chat and prompt in Portuguese. Nevertheless, the answers must be in
    English.

# Tests

- Never execute tests automatically.
- Ask me if I prefer to test by hand.

# Code Style

- 80-column lines
- Indentation:
    - 4 spaces
- Comments:
    - only before blocks or functions, never in between lines
    - if a line requires comments, create an explicit block for the related
      lines and comment on top
- Ifs:
    - always use else (unless empty), even if the true case returns
    - both paths should be clear with proper indentation
    - In C:

```c
if (...) {
    ...
    return ...;
} else {
    ...     // indent even with return above
}
```
