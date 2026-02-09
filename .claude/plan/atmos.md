# PR 2: Atmos in the Browser

## Goal
Compile and run `.atm` code entirely in the browser.

## PR
- https://github.com/atmos-lang/web/pull/2
- Branch: `atmos-wasmoon`

## Tasks
- [x] Copy atmos-lang/atmos compiler Lua sources to lua/
- [x] Copy lua-atmos/atmos runtime Lua sources to lua/
- [x] Copy f-streams library to lua/
- [x] Load all modules via package.preload in wasmoon
- [x] Compile .atm source with atm_loadstring
- [x] Execute compiled Lua with atmos.call
- [x] Adapt atmos.env.clock for browser (Date.now())
- [x] UI: editor for .atm code + output + Run button
- [x] Default: Hello World with watching/every

## Architecture
- lua/ dir contains all Lua sources (16 files)
- JS fetches all files, registers in package.preload
- Compilation: atm source -> atm_loadstring -> Lua function
- Execution: atmos.call(f) runs the event loop synchronously
- Clock: now_ms() global injected from JS (Date.now())

## Pending
- Test in browser (user should verify)
- Merge PR
- Known limitation: synchronous execution blocks UI
