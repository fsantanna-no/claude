# Plan: loop

## Goal

Add a `do_exit` parameter to `event_from_sdl` to control whether
`SDL_QUIT` calls `exit(0)` or simply returns. This prepares the
ground for a future `pico_input_loop` function that needs quit
events to propagate back to the caller instead of terminating.

## Repository

- [work] https://github.com/fsantanna/pico-sdl
- File: `src/pico.c`

## Changes

### 1. `event_from_sdl` — add `do_exit` parameter
- **File:** `src/pico.c:1243`
- **Current:**
  `static int event_from_sdl (Pico_Event* e, int xp)`
- **New:**
  `static int event_from_sdl (Pico_Event* e, int xp, int do_exit)`

### 2. `SDL_QUIT` case — use `do_exit`
- **File:** `src/pico.c:1245-1249`
- **Current:**
  ```c
  case SDL_QUIT: {
      if (!S.expert) {
          exit(0);
      }
      break;
  }
  ```
- **New:**
  ```c
  case SDL_QUIT: {
      if (!S.expert && do_exit) {
          exit(0);
      }
      break;
  }
  ```

### 3. Update all callers — pass `do_exit=1`
- `pico_input_delay`        (line 1403): `event_from_sdl(&e, SDL_ANY, 1)`
- `pico_input_event`        (line 1418): `event_from_sdl(&x, type, 1)`
- `pico_input_event_ask`    (line 1430): `event_from_sdl(evt, type, 1)`
- `pico_input_event_timeout`(line 1439): `event_from_sdl(evt, type, 1)`

### 4. (Future) `pico_input_loop` — will pass `do_exit=0`
- Not yet implemented; will call `event_from_sdl(..., 0)` so
  `SDL_QUIT` propagates as a regular event instead of terminating.

## Workflow

- All code commits go to `pico-sdl` repo (new branch → PR)
- Plan file also committed to `pico-sdl` repo

## Status

- [x] Discuss parameter name (`do_exit` chosen)
- [ ] Copy plan file into `pico-sdl` repo
- [ ] Apply edits to `src/pico.c`
- [ ] Commit & push to new branch in `pico-sdl`
- [ ] Create PR
