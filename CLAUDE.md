# Code Style

- 80-column lines
- Indentation:
    - 4 spaces
- Comments:
    - only before blocks or functions, never in between lines
    - if a line requires comments, create an explicit block for the related
      lines and comment on top

# Git Worktree

If asked to create a worktree:

```
> create worktree <name>
```

Substitute <name> as follows:

```
git worktree add ../<name> -b ai/<name>
```

Then, to remove a worktree:

```
> remove worktree <name>
```

Substitute <name> as follows:

```
git worktree remove ../<name>
git branch -D ai/<name>         # if above succeeds
```

# Extra

- Never execute tests. Let me do this step by hand.
