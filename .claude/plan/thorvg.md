# ThorVG Integration into pico-sdl

## Status: ANALYSIS / PLAN MODE

---

## 0. Context: pico-sdl Rendering Architecture

pico-sdl uses `SDL_Renderer` for ALL drawing. In test mode
(`PICO_TESTS`) it explicitly uses `SDL_RENDERER_SOFTWARE`.
In production it uses `SDL_RENDERER_ACCELERATED`.

All primitives go through the SDL_Renderer API:
- `SDL_RenderFillRect`, `SDL_RenderDrawLine`, etc. (SDL2)
- `filledTrigonRGBA`, `filledEllipseRGBA`, etc. (SDL2_gfx)

The default canvas is **100x100 logical pixels**. At this size
CPU rasterization is trivially fast -- the GPU vs CPU
distinction is irrelevant for performance.

The library draws to an off-screen `SDL_Texture` (`G.main.tex`)
via `SDL_SetRenderTarget`, then composites to the window.
Layers are additional `SDL_Texture` targets.

**Key insight**: since pico-sdl already works with SW rendering
and targets small canvases, ThorVG's CPU rasterization is NOT
a performance downgrade. Both engines end up doing CPU work.
This makes full replacement of SDL2_gfx viable.

---

## 1. SVG Support

### How
```
SVG file --> ThorVG SwCanvas --> uint32_t[] buffer
         --> SDL_CreateTexture (ARGB8888)
         --> rendered as a layer (like images)
```

Auto-detect by extension in `pico_output_draw_image()`:
`.svg` routes to ThorVG, `.png`/`.jpg` routes to SDL2_image.

### Pros
- SVG becomes just another image format -- reuses layer/hash
- Scalable to any resolution without pixelation
- ThorVG's SVG parser is built-in (no libxml)
- Enables vector art, icons, logos

### Cons
- Adds ~150KB binary size
- CPU-intensive on first rasterization (cached after)
- C++ link dependency (`-lstdc++`)

### Verdict: **YES** -- low risk, high value

---

## 2. Replace SDL2 Primitives

### Current deps for primitives

| What           | Library   | Provides                    |
|----------------|-----------|------------------------------|
| Rect, Line, Pt | SDL2      | `SDL_RenderFillRect`, etc.  |
| Tri, Oval, Poly| SDL2_gfx  | `filledTrigonRGBA`, etc.    |

SDL2_gfx provides only: aliased rendering, 1px stroke,
solid fill, no transforms.

### ThorVG replacement

ThorVG renders ALL primitives (rect, line, tri, oval, poly,
path) to a `uint32_t[]` buffer via SwCanvas. The buffer is
uploaded to an `SDL_Texture` and composited.

Since pico-sdl already uses SW rendering (tests) and has
small canvases (100x100 default), the CPU cost is negligible.

### What changes
- Drop SDL2_gfx dependency entirely
- Drop SDL2 native drawing calls (`SDL_RenderFillRect`, etc.)
- All primitives rendered by ThorVG into a buffer
- Buffer uploaded as texture per present cycle

### What stays
- SDL2 for: windowing, input, audio, texture management
- SDL2_image for: PNG/JPG loading
- SDL2_ttf for: text (unless also replaced, see section 6)
- SDL2_mixer for: audio

### Verdict: **YES** -- viable given SW rendering + small canvas

---

## 3. Pros and Cons (Full Replacement)

### Pros

**Quality gains:**
- Anti-aliased rendering on all primitives (SDL2_gfx = aliased)
- Rounded rectangles (`append_rect(rx, ry)`)
- Gradient fills (linear, radial) on any shape
- Bezier curves / arbitrary vector paths
- Variable stroke width, cap, join, dash
- Per-object rotation, flip, scale (impossible with SDL2_gfx)
- Per-object opacity and blend modes

**Architectural gains:**
- Drop 1 dependency (SDL2_gfx)
- Single rendering engine for everything (shapes + SVG + Lottie)
- Consistent quality across all drawing operations
- Scene graph enables composition, masking, effects

**New capabilities:**
- SVG rendering
- Lottie animations
- Masking, clipping by arbitrary shapes
- Blur, drop shadow, tint effects

### Cons

**Architecture:**
- Immediate mode mismatch: pico-sdl draws one primitive at a
  time; ThorVG builds a scene graph then renders all at once.
  Options:
  - (a) Render each primitive individually (simple, wasteful)
  - (b) Batch primitives per present cycle (efficient, complex)
  - (c) Maintain a persistent ThorVG canvas sized to the
    logical texture, render into it per-draw or per-present
- Buffer upload overhead: `SDL_UpdateTexture()` per present.
  At 100x100 = 40KB, this is negligible. At 1000x1000 = 4MB,
  still fast for CPU.

**Build:**
- ThorVG is C++, pico-sdl is C. Requires `-lstdc++` at link.
- ThorVG uses Meson; pico-sdl uses Makefile.
  Must pre-build or use pkg-config.
- ~150KB added to binary.

**Rendering model:**
- SDL2 native rect/line/point are extremely fast (direct
  framebuffer writes in SW mode). ThorVG adds vector path
  overhead even for trivial shapes. For 100x100 canvas this
  doesn't matter, but it's architecturally heavier.
- ThorVG renders to its own buffer, NOT to `SDL_Texture`
  directly. All drawing must go through the buffer->texture
  upload path. This means pico-sdl can't mix ThorVG and SDL2
  drawing on the same texture without extra steps.

**Text:**
- If only primitives are replaced (not text), two rendering
  engines coexist: ThorVG for shapes, SDL_ttf for text.
  The present cycle must composite both.

### Summary: pros outweigh cons for an educational library

The quality and feature gains are substantial. The performance
costs are negligible at educational canvas sizes. The main
complexity is the immediate-mode-to-scene-graph adaptation.

---

## 4. Effect on Raster-Based Operations

| Component | Impact    | Why                              |
|-----------|-----------|----------------------------------|
| Layers    | None      | Layers remain `SDL_Texture`;     |
|           |           | ThorVG output = new layer source |
| Images    | None      | PNG/JPG still via SDL2_image     |
| Buffers   | None      | `Pico_Color_A[]` unchanged;      |
|           |           | ThorVG output IS an RGBA buffer  |
| Video     | None      | Y4M/YUV pipeline untouched       |
| Clear     | None      | `SDL_RenderClear` unchanged      |
| Push/Pop  | None      | State stack unchanged            |

ThorVG adds a new source of pixels (vector rendering) that
feeds into the existing texture/layer system. All raster
operations continue unchanged.

---

## 5. New Capabilities (Not Currently Possible)

| Capability            | Value  | ThorVG feature                  |
|-----------------------|--------|---------------------------------|
| SVG images            | High   | `tvg_picture_load("file.svg")`  |
| Lottie animations     | High   | `tvg_picture_load("anim.json")` |
| Rounded rectangles    | Medium | `tvg_shape_append_rect(rx, ry)` |
| Gradient fills        | Medium | Linear + radial on any shape    |
| Bezier paths          | Medium | `tvg_shape_cubic_to()`          |
| Anti-aliased shapes   | Medium | Built-in sub-pixel AA           |
| Variable stroke width | Medium | `tvg_shape_set_stroke_width()`  |
| Stroke cap/join/dash  | Low    | Round, bevel, dash patterns     |
| Per-shape transforms  | High   | Rotate/flip/scale any primitive |
| Per-shape opacity     | Medium | Independent of global alpha     |
| Blend modes (16)      | Medium | Multiply, screen, overlay, ...  |
| Shape masking         | Low    | Alpha/luma masks on any paint   |
| Shape clipping        | Low    | Clip by arbitrary path          |
| Blur / drop shadow    | Low    | Scene-level post-processing     |
| TVG binary format     | Low    | Compact pre-compiled vectors    |

---

## 6. Text / Vector Fonts

### Current: SDL_ttf + FreeType
- Renders glyphs as bitmaps (pixel-hinted)
- Excellent small-text legibility (hinting aligns to pixel grid)
- Cached as `SDL_Texture` via layer hash
- Supports kerning, ligatures (via HarfBuzz), complex scripts
- 20+ years mature

### ThorVG text: glyphs as vector paths
- Built-in TTF loader (no FreeType, no SDL_ttf dependency)
- Converts glyph outlines to ThorVG vector paths
- Rasterized by ThorVG's general-purpose engine

### Are there better vector font formats?

**No.** TTF IS the standard vector font format. The outlines
in TTF files are already Bezier curves (quadratic). ThorVG
reads these outlines and converts them to its own cubic Bezier
paths. OTF adds CFF (cubic Bezier) outlines but ThorVG doesn't
support OTF yet (planned v2.0). SVG fonts are deprecated.

The font format is not the issue. The issue is **hinting**.

### Quality comparison

| Aspect              | SDL_ttf (FreeType) | ThorVG text        |
|---------------------|--------------------|--------------------|
| Small text (8-14px) | Excellent (hinted) | Blurry (no hinting)|
| Large text (24px+)  | Good               | Excellent          |
| HiDPI displays      | Good               | Excellent          |
| Gradient fill       | Impossible         | Supported          |
| Stroke/outline      | Impossible         | Supported          |
| Rotation/scale      | Impossible         | Supported          |
| Per-glyph effects   | Impossible         | Supported          |
| Kerning             | Yes                | Not yet            |
| Ligatures           | Yes (HarfBuzz)     | Not yet            |
| Complex scripts     | Yes                | Not yet            |

### Verdict

**For an educational library with small canvases (100x100):**
- Text is typically large relative to canvas = ThorVG is fine
- Hinting matters less when pixels are large/zoomed
- Gradient/stroke/rotation on text is a real feature gain
- BUT: ThorVG text is young, no kerning, no complex scripts

**Recommendation: REPLACE SDL_ttf with ThorVG text.**
The educational context (large text, simple scripts) plays
to ThorVG's strengths. Gradient and rotation on text are
genuine new capabilities. pico-sdl already uses an embedded
`tiny_ttf.h` (stb-style), so the lack of hinting is
consistent with the existing fallback approach.

This would also drop SDL_ttf as a dependency.

---

## 7. Rotation, Flip, Scaling for Primitives

### Current situation

| Object    | Rotation | Flip | Scale |
|-----------|----------|------|-------|
| Layers    | Yes      | Yes  | Yes   |
| Rect      | No       | No   | No    |
| Line      | No       | No   | No    |
| Tri       | No       | No   | No    |
| Oval      | No       | No   | No    |
| Poly      | No       | No   | No    |
| Text      | No       | No   | No    |
| Image     | No       | No   | No*   |

*Images scale via destination rect, but don't support
rotation or flip independently.

SDL2_gfx takes absolute pixel coordinates. There is no
transform parameter. Impossible to rotate a triangle.

### With ThorVG: every paint object gets transforms

```c
tvg_paint_translate(shape, tx, ty);
tvg_paint_rotate(shape, degrees);
tvg_paint_scale(shape, factor);

// OR full 3x3 affine matrix:
Tvg_Matrix m = { cos,-sin,tx, sin,cos,ty, 0,0,1 };
tvg_paint_set_transform(shape, &m);

// Flip = negative scale:
// H-flip: scale(-1, 1) + translate(width, 0)
// V-flip: scale(1, -1) + translate(0, height)
```

### How to expose in pico-sdl

Rotation already exists as state (`S.rotation`). It currently
only applies to layers via `SDL_RenderCopyEx`. With ThorVG,
the same state applies to all primitives:

```c
pico_set_rotation(45);
pico_output_draw_rect(rect);  // rotated 45 degrees
```

No new API needed -- existing `pico_set_rotation` and
`pico_set_flip` simply apply to ThorVG paint objects.

### Verdict: **major win** -- unifies transform behavior

---

## 8. Other Properties

### Properties ThorVG adds to every paint object

| Property      | Current (SDL2) | With ThorVG         |
|---------------|----------------|---------------------|
| Rotation      | Layers only    | **Any primitive**   |
| Flip          | Layers only    | **Any primitive**   |
| Scale         | Layers only    | **Any primitive**   |
| Opacity       | Global only    | **Per-primitive**   |
| Blend mode    | 1 (alpha)      | **16 modes**        |
| Fill          | Solid only     | **Solid + gradient**|
| Fill rule     | N/A            | **Even-odd / NZ**   |
| Stroke width  | 1px            | **Any float**       |
| Stroke cap    | N/A            | **Butt/round/sq**   |
| Stroke join   | N/A            | **Miter/round/bvl** |
| Stroke dash   | N/A            | **Arbitrary**       |
| Clipping      | Rect only      | **Any shape**       |
| Masking       | None           | **10 methods**      |
| Anti-aliasing | None           | **Sub-pixel AA**    |
| Effects       | None           | **Blur/shadow/tint**|

### Which to expose in pico-sdl?

**Phase 1 (immediate, via state):**
- `pico_set_stroke_width(float)` -- already issue #62
- `pico_set_rotation(float)` -- extend to all primitives
- `pico_set_flip(int)` -- extend to all primitives

**Phase 2 (new features):**
- Gradient fills via `pico_set_color_gradient_*()`
- Rounded rects via radius param on `pico_output_draw_rect()`
- Bezier paths via `pico_output_draw_path()`

**Later:**
- Blend modes, masking, effects (advanced, less educational)

---

## 9. How Exactly to Integrate ThorVG with SDL2

### 9.1 Colorspace

```
TVG_COLORSPACE_ARGB8888S  =  SDL_PIXELFORMAT_ARGB8888
```
Use straight alpha (`S` suffix). Byte-compatible with SDL.
Straight alpha works with `SDL_BLENDMODE_BLEND` directly.
Slight perf cost vs premultiplied, but simpler integration.

### 9.2 Architecture Options

**Option A: ThorVG as primary renderer (RECOMMENDED)**

Replace the SDL_Renderer drawing path entirely for shapes.
ThorVG renders into a buffer the size of the logical texture.
The buffer is uploaded as `SDL_Texture` per present cycle.

```
                    pico-sdl
                    ┌──────────────────────┐
                    │   Logical Texture     │
                    │   (100x100 default)   │
                    └──────────┬───────────┘
                               │
              ┌────────────────┼──────────┐
              │                │          │
        ┌─────┴─────┐   ┌─────┴───┐  ┌───┴────┐
        │  ThorVG   │   │ SDL2_img│  │ Video  │
        │  SwCanvas │   │ PNG/JPG │  │ Y4M    │
        │           │   └─────────┘  └────────┘
        │ shapes    │
        │ text      │
        │ SVG       │
        │ Lottie    │
        └─────┬─────┘
              │
        uint32_t[] buffer (100x100 = 40KB)
              │
        SDL_UpdateTexture()
              │
        SDL_RenderCopy() to window
```

Flow:
1. `pico_init`: create ThorVG SwCanvas, allocate buffer
2. Each `pico_output_draw_*`: add/modify ThorVG paint objects
3. `_pico_output_present`:
   - `tvg_canvas_update()` + `tvg_canvas_draw()` + `tvg_canvas_sync()`
   - `SDL_UpdateTexture()` to upload buffer
   - `SDL_RenderCopy()` to window (handles phy/log scaling)
4. `pico_close`: destroy canvas, free buffer

**Option B: ThorVG as supplementary renderer**

Keep SDL_Renderer for simple shapes (rect, line, point).
Use ThorVG only for features SDL2 can't do (SVG, gradients,
AA shapes, paths). Two rendering paths coexist.

Pro: less change. Con: inconsistent quality, complex present.

### 9.3 Immediate Mode Adaptation

pico-sdl's immediate mode calls `_pico_output_present(0)`
after each draw call (non-expert mode). With ThorVG:

**Approach: rebuild scene per present**

```c
// Each draw call adds a paint to the canvas:
void pico_output_draw_rect (Pico_Rel_Rect* r) {
    Pico_Abs_Rect abs = pico_cv_rect_rel_abs(r);
    Tvg_Paint* shape = tvg_shape_new();
    tvg_shape_append_rect(shape, abs.x, abs.y,
                          abs.w, abs.h, S.rx, S.ry);
    // Apply current state
    tvg_shape_set_fill_color(shape, S.color.r,
                             S.color.g, S.color.b, S.alpha);
    if (S.rotation != 0) {
        tvg_paint_rotate(shape, S.rotation);
    }
    tvg_canvas_push(G.tvg_canvas, shape);

    _pico_output_present(0);  // triggers render
}
```

```c
// Present: render all accumulated paints
void _pico_output_present (int
) {
    tvg_canvas_update(G.tvg_canvas);
    tvg_canvas_draw(G.tvg_canvas, true);  // clear + render
    tvg_canvas_sync(G.tvg_canvas);

    SDL_UpdateTexture(G.main.tex, NULL, G.tvg_buffer,
                      G.tvg_w * sizeof(uint32_t));

    // existing present logic (scale to window, grid, etc.)
    SDL_SetRenderTarget(G.ren, NULL);
    SDL_RenderCopy(G.ren, G.main.tex, &src, &dst);
    _show_grid();
    SDL_RenderPresent(G.ren);
    SDL_SetRenderTarget(G.ren, G.main.tex);
}
```

**Issue: accumulated state between presents.**
Each `_pico_output_present` must re-render ALL paints drawn
so far (since the canvas is cleared). Options:
- (a) Keep all paints in the canvas across presents
  (canvas grows, `clear=false` to preserve)
- (b) Render each primitive independently
  (wasteful but matches immediate model)
- (c) Maintain a "committed" buffer and a "pending" buffer
  (committed = screenshot of all previous draws)

Option (a) is cleanest: never clear the canvas, only add.
On `pico_output_clear()`, remove all paints and clear.

### 9.4 Persistent Canvas Pattern

```c
// pico_output_clear:
tvg_canvas_remove(G.tvg_canvas, NULL); // remove all paints
// draw background color shape

// pico_output_draw_rect:
Tvg_Paint* shape = tvg_shape_new();
// ... configure ...
tvg_canvas_push(G.tvg_canvas, shape);

// _pico_output_present:
tvg_canvas_update(G.tvg_canvas);
tvg_canvas_draw(G.tvg_canvas, false); // DON'T clear buffer
tvg_canvas_sync(G.tvg_canvas);
SDL_UpdateTexture(...);
```

With `draw(false)` (no clear), ThorVG only re-renders dirty
regions. Previously drawn shapes remain in the buffer.
This matches pico-sdl's single-buffer immediate model.

### 9.5 Layer Integration

Each layer gets its own ThorVG canvas + buffer:

```c
typedef struct {
    SDL_Texture* tex;
    Tvg_Canvas*  tvg;       // per-layer ThorVG canvas
    uint32_t*    tvg_buf;   // per-layer buffer
    // ... existing fields ...
} Pico_Layer;
```

When `pico_set_layer(name)` switches the active layer,
the active ThorVG canvas switches too. Drawing goes to
that layer's canvas/buffer.

### 9.6 Build Integration

```makefile
# Build ThorVG as static lib (one-time):
#   cd thorvg
#   meson setup build -Ddefault_library=static \
#     -Dbindings=capi -Dloaders=svg,lottie,ttf \
#     -Dengines=sw -Dsavers=
#   ninja -C build

THORVG_INC = -I./thorvg/inc
THORVG_LIB = -L./thorvg/build/src -lthorvg -lstdc++

CFLAGS  += $(THORVG_INC)
LDFLAGS += $(THORVG_LIB)
```

### 9.7 Dependencies After Integration

| Before            | After              |
|-------------------|--------------------|
| SDL2              | SDL2               |
| SDL2_gfx          | --REMOVED--        |
| SDL2_ttf          | --REMOVED-- (opt.) |
| SDL2_image        | SDL2_image         |
| SDL2_mixer        | SDL2_mixer         |
| --                | ThorVG (static)    |

Net: -1 or -2 deps, +1 dep. Total deps same or fewer.

---

## 10. Summary

| # | Topic                  | Verdict                         |
|---|------------------------|---------------------------------|
| 1 | SVG support            | YES -- add as image format      |
| 2 | Replace SDL2 prims     | YES -- viable with SW rendering |
| 3 | Pros/cons              | Pros outweigh at 100x100 canvas |
| 4 | Raster ops impact      | None -- layers/images/video OK  |
| 5 | New capabilities       | 15+ features unlocked           |
| 6 | Text / vector fonts    | YES replace -- better for edu   |
| 7 | Primitive transforms   | YES -- major win, unifies API   |
| 8 | Other properties       | Gradients, stroke, blend, AA    |
| 9 | SDL2 integration       | ThorVG as primary renderer      |

### Recommended: Option A (full replacement)

Replace SDL_Renderer drawing with ThorVG SwCanvas:
- ThorVG renders shapes, text, SVG, Lottie to buffer
- SDL2 handles windowing, input, audio, texture display
- Drop SDL2_gfx (and optionally SDL2_ttf)
- Persistent canvas with `draw(false)` for immediate mode

---

## Pending Actions

- [ ] User decision: Option A (full) or Option B (supplementary)
- [ ] Prototype: ThorVG buffer -> SDL_Texture at 100x100
- [ ] Prototype: immediate mode with persistent canvas
- [ ] Test: text quality at pico-sdl's typical sizes
- [ ] Build: compile ThorVG as static lib with CAPI
