# Plan: Atmos in the Browser

## Goal
Run Atmos from the browser using wasmoon (Lua 5.4 WASM).

## Phases

### Phase 1: Lua in the Browser (CURRENT)
- [ ] Set up simplest possible HTML page with wasmoon
- [ ] Load Lua 5.4 VM in browser
- [ ] Run a basic Lua script (print "hello")
- [ ] Verify `<close>` works in the VM

### Phase 2: Atmos Compiler in the Browser
- [ ] Clone/fetch atmos-lang/atmos compiler sources (lexer, parser, coder, etc.)
- [ ] Load compiler Lua files into wasmoon VM
- [ ] Compile a simple `.atm` program to Lua inside the browser

### Phase 3: Atmos Runtime in the Browser
- [ ] Clone/fetch lua-atmos/atmos runtime
- [ ] Load runtime into the VM
- [ ] Run compiled Atmos Lua output with the runtime

### Phase 4: Clock Environment (Browser)
- [ ] Adapt `atmos.env.clock` for browser (requestAnimationFrame / setTimeout)
- [ ] Run the "Hello World" Atmos example with timing

### Phase 5: UI
- [ ] Text editor area for `.atm` code
- [ ] Output area
- [ ] "Run" button

## Notes
- wasmoon bundles official Lua 5.4 C VM compiled to WASM
- `<close>` and `coroutine.close()` are essential for Atmos
- Keep it as simple as possible — plain HTML, no bundler
