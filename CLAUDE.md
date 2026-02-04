- Update `CLAUDE.md` when necessary.
    - global at `~/.claude/CLAUDE.md`
    - local at `.claude/CLAUDE.md`

- As soon as `claude` starts, it should permanently change the terminal title
  to `"[CC] <current directory>"`.

# Language

- **English**: All answers, comments, and any other kind of output should be in
    English.
- I may chat and prompt in Portuguese. Nevertheless, the answers must be in
    English.

# Persistent Plans

A repository can save persistent plans in `.claude/*`, e.g.:

- `.claude/feature.md`

Plan may be mentioned and should be updated during sessions.

# Plan Mode Workflow

- Starts in Plan Mode.
- Always enter plan mode before making any file edits.
- Never edit files without explicit user authorization.
- Just before edit, you should make one last suggestion.
- After completing authorized edits
    - print a summary of places changed:
        - "<file>:<line> | place | description"
        - place can be a function, data structure, etc
        - use 1 line fore each change
    - return to plan mode automatically
        - **take this step very seriously**
        - show a message "I'm back to plan mode..."
- Remain in Plan Mode.

# Tests

- Never execute tests.
- Let me do this step by hand.

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
    ...     // indent event with return above
}
```
