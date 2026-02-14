# TODO

1. Remove Python dependency from build process
   - `build.py` currently generates `index-standalone.html`
   - Replace with a shell script or plain JS build
2. Copy Lua files from upstream repos instead of hard-coding
   - Currently `lua/` contains manually copied files
   - Should fetch from `atmos-lang/atmos` and `lua-atmos/atmos`
     repos at specific tags/versions
   - Use git tags (e.g., `v0.5`) for reproducible builds
