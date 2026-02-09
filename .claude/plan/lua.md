# PR 1: Lua in the Browser

## Goal
Simplest possible page running Lua 5.4 via wasmoon.

## Tasks
- [ ] Create `index.html` with wasmoon from CDN
- [ ] Initialize Lua 5.4 VM in JavaScript
- [ ] Redirect Lua `print` to a `<pre>` output area
- [ ] Run a basic Lua script (print "hello")
- [ ] Verify `<close>` works (to-be-closed variables)
- [ ] Verify `coroutine.close()` works
- [ ] Minimal UI: textarea for Lua code + "Run" button + output

## Constraints
- Single `index.html` file, no bundler
- Plain HTML + inline JS
- Load wasmoon from CDN (unpkg or jsdelivr)

## Pending
- (nothing yet)
