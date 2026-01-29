# pico-lua

`pico-lua` is a Lua binding for `pico-sdl`, a graphics library for 2D games and
applications based on SDL.

- `pico-sdl`: https://github.com/fsantanna/pico-sdl/
- `pico-lua`: https://github.com/fsantanna/pico-sdl/tree/main/lua/
- API: https://github.com/fsantanna/pico-sdl/blob/main/lua/doc/api.md

# Idioms and Caveats:

- Color values:
    - use r,g,b fields
    - `@{ r=0xCC, g=0x11, b=77 }`
