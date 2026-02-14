# TODO

1. Remove Python dependency from build process
   - `build.py` currently generates `index-standalone.html`
   - Replace with a shell script or plain JS build
2. Copy Lua files from upstream repos instead of hard-coding
   - Currently `lua/` contains manually copied files
   - Should fetch from `atmos-lang/atmos` and `lua-atmos/atmos`
     repos at specific tags/versions
   - Use git tags (e.g., `v0.5`) for reproducible builds
3. Reorganize repo structure
   - `web/` — all deployable files (root for GitHub Pages)
   - `web/lua/index.html` — Lua REPL without Atmos (PR 1)
   - `web/atmos/index.html` — Atmos REPL (PR 2)
   - Lua sources under `web/atmos/lua/` (or shared)
