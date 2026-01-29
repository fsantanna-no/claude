# Atmos: The Programming Language

- GitHub: https://github.com/atmos-lang/atmos/
- Manual: https://github.com/atmos-lang/atmos/blob/main/doc/manual-out.md

Atmos is an experimental programming language that compiles to Lua 5.4 and
reconciles three paradigms:

1. Structured Concurrency: deterministic task scheduling with safe termination
2. Event-Driven Programming: `await`/`emit` primitives for reactive behavior
3. Functional Streams: lazy list combinators for infinite sequences

- Core constructs:
  - `val`/`var`/`set` for declarations and assignment
  - `await(:tag)` / `emit(:tag)` for events
  - `par`, `par_or`, `par_and` for parallel composition
  - `spawn`, `task`, `tasks` for concurrency
  - hierarchical tags with `:` prefix (e.g., `:Event.Click`)

# Idioms and Caveats:

- No precendence for binary operators:
    - use parenthesis to disambiguate
    - `1 + 2*3` becomes `1 + (2*3)`
- Statements as expressions:
    - even complex statements evaluate to final value(s)
    - `val x,y = match ... { ... }`
- Pattern matching:
    - `match` if possible
        - avoid `else` if possible to exhaust cases
    - `ifs` as second option, if 2+ conditions
    - `if` as third option
- Tasks
    - `pub` is a special public variable for tasks
        - use `set pub = ...` inside
        - use `t.pub = ...` outside
    - `spawn`:
        - do not use pools (`tasks`) for singleton instances
