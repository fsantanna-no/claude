- pico-atmos

See `pico-lua.md` and `atmos.md`.

`pico-atmos` combines Atmos and `pico-lua` to program 2D applications and
games.

- Games developed in `pico-atmos`:
    - Birds: https://github.com/atmos-lang/pico-birds/
        - a tutorial progression from `01` to `11`
    - Rocks: https://github.com/atmos-lang/pico-rocks/
        - a simple, but complete game

# Idioms and Caveats:

- `pico.set.*` becomes `pico.zet.*` (because `set` is a keyword in Atmos)
- Key events:
    - `every it in :key.dn { ... }`
    - `it.key` can be `:Left`, `:Up`, etc
- Collisions
    - typically pass `t[i].pub.rect` to `pico.vs`
