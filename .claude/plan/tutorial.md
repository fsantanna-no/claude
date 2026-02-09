# PR 3: Try Atmos Tutorial

## Goal
Interactive step-by-step tutorial, like "Try Ceu".

## Reference: Try Ceu (http://www.ceu-lang.org/try.php)
- 2x2 grid layout (lesson | code / results | input)
- 16 lessons with prev/next navigation + index modal
- Each lesson: `.ceu` code + `_in.ceu` input + `.html` explanation
- Plain textarea editor, no syntax highlighting
- "Run" button sends code to backend for compilation
- Our version: everything runs client-side (wasmoon)

## Layout (adapted for Atmos)
```
+-------------------------------+-------------------------------+
|         LESSON PANEL          |         CODE PANEL            |
|  [Index] [<] [N] [>]         |  [Reset] [Run]                |
|                               |                               |
|  <Title + HTML explanation>   |  <textarea for .atm code>    |
+-------------------------------+-------------------------------+
|                  OUTPUT PANEL (full width)                     |
|  <pre> output from running the program </pre>                 |
+---------------------------------------------------------------+
```
Note: no Input panel needed (Atmos uses internal events,
not external text input like Ceu).

## Tasks
- [ ] Design tutorial progression (topics and examples)
- [ ] Create lesson data (JSON: code + HTML explanation)
- [ ] UI: 2-panel top + output bottom layout
- [ ] UI: step navigation (prev/next + index)
- [ ] UI: lesson panel with title + explanation
- [ ] UI: code editor with pre-filled example
- [ ] UI: Reset (reload original) + Run buttons
- [ ] UI: output area showing print results
- [ ] Allow user to edit and re-run examples
- [ ] Steps covering:
      1. Welcome / UI intro
      2. Hello World (print, basics)
      3. Variables (val, var, set)
      4. Tasks and spawn
      5. await / emit
      6. watching / every
      7. par_or / par_and
      8. defer
      9. Tags and events
      10. Streams (experimental)

## Dependencies
- PR 2 (Atmos running in the browser)

## Pending
- Finalize exact lesson content and examples
