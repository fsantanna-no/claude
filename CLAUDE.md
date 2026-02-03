- As soon as `claude` starts, it should permanently change the terminal title
  to `"[CC] <current directory>"`.
- Update `CLAUDE.md` when necessary.

# Plan Mode Workflow

- Starts in Plan Mode.
- Always enter plan mode before making any file edits.
- Never edit files without explicit user authorization.
- Just before edit, you should make one last suggestion.
- After completing authorized edits
    - print a summary of places changed:
        - "<file>:<line> | place | description"
        - place can be a function, data structure, etc
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
