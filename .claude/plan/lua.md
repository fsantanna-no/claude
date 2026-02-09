# PR 1: Lua in the Browser

## Goal
Simplest possible page running Lua 5.4 via wasmoon.

## PR
- https://github.com/atmos-lang/web/pull/1
- Branch: `lua-wasmoon`

## Tasks
- [x] Create `index.html` with wasmoon from CDN (jsdelivr)
- [x] Initialize Lua 5.4 VM in JavaScript
- [x] Redirect Lua `print` to a `<pre>` output area
- [x] Run a basic Lua script (print "hello")
- [x] Verify `<close>` works (to-be-closed variables)
- [x] Verify `coroutine.close()` works
- [x] Minimal UI: textarea for Lua code + "Run" button + output

## Constraints
- Single `index.html` file, no bundler
- Plain HTML + inline JS
- Load wasmoon from CDN (jsdelivr +esm)

## Pending
- Test in browser (user should verify)
- Merge PR
