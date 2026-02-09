# PR 2: Atmos in the Browser

## Goal
Compile and run `.atm` code entirely in the browser.

## Tasks
- [ ] Fetch atmos-lang/atmos compiler Lua sources
- [ ] Fetch lua-atmos/atmos runtime Lua sources
- [ ] Load compiler into wasmoon VM (lexer, parser, coder)
- [ ] Load runtime into wasmoon VM
- [ ] Compile `.atm` source to Lua inside the browser
- [ ] Execute compiled Lua with the runtime
- [ ] Adapt `atmos.env.clock` for browser
      (requestAnimationFrame / setTimeout)
- [ ] UI: editor for `.atm` code + output + "Run" button
- [ ] Test with Atmos "Hello World" example

## Dependencies
- PR 1 (Lua in the browser via wasmoon)

## Pending
- (nothing yet)
