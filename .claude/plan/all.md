# Plan: Atmos in the Browser

## Goal
Run Atmos from the browser using wasmoon (Lua 5.4 WASM).

## PRs (one each)

### PR 1: Lua in the Browser (`lua.md`)
- Simplest possible HTML page with wasmoon
- Lua 5.4 VM running in the browser
- Verify `<close>` and coroutines work

### PR 2: Atmos in the Browser (`atmos.md`)
- Load Atmos compiler (lexer, parser, coder) into wasmoon
- Load lua-atmos runtime into wasmoon
- Compile and run `.atm` code in the browser
- Browser-compatible clock environment

### PR 3: Try Atmos Tutorial (`tutorial.md`)
- Interactive step-by-step tutorial (like "Try Ceu")
- Guided examples with explanations
- Editor + output for each step
- Progression from basics to concurrency

### PR 4: Pico-Atmos Integration (`pico.md`)
- Canvas-based game environment in the browser
- Adapt pico-atmos for browser rendering
- Input events (keyboard, mouse) bridged to Atmos
- Demo games (birds, rocks, etc.)

## Notes
- wasmoon bundles official Lua 5.4 C VM compiled to WASM
- `<close>` and `coroutine.close()` are essential for Atmos
- Keep it as simple as possible — plain HTML, no bundler
- Each PR builds on the previous one
