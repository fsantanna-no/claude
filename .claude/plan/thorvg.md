# ThorVG Integration into pico-sdl

## Status: ANALYSIS / PLAN MODE

---

## 1. What is ThorVG?

- Lightweight C++ vector graphics engine (~150KB binary)
- MIT license, zero mandatory dependencies
- Powers Tizen OS, Godot Engine, LVGL, dotLottie
- Rendering backends: Software (CPU), OpenGL, WebGPU
- Input formats: SVG, Lottie (animations), TVG (compact binary)
- Has a complete **C API** (`thorvg_capi.h`) with opaque types
- Built-in TTF font rendering (no FreeType needed)
- Built-in SVG parser (no libxml needed)
- Renders to a `uint32_t*` ARGB8888 buffer (SW backend)

---

## 2. Current pico-sdl Rendering Architecture

| Component       | Library      | Usage                          |
|-----------------|-------------|--------------------------------|
| Rect, Line, Pt  | SDL2 native | `SDL_RenderFillRect`, etc.     |
| Tri, Oval, Poly | SDL2_gfx    | `filledTrigonRGBA`, etc.       |
| Images          | SDL2_image  | `IMG_LoadTexture` (PNG/JPG)    |
| Text            | SDL2_ttf    | `TTF_RenderText_Solid`         |
| Video           | custom      | Y4M parser + `SDL_UpdateYUV`   |
| Layers/Buffers  | SDL2 native | `SDL_CreateTexture` + targets  |

All rendering goes through `SDL_Renderer` (hardware-accelerated or
software). Drawing happens to off-screen `SDL_Texture` targets
(layers), then composited to the window.

---

## 3. Integration Approach: SVG Support (Additive)

### How it would work

```
SVG file --> ThorVG SwCanvas --> uint32_t[] buffer
         --> SDL_CreateTexture (ARGB8888)
         --> rendered as a layer (like images)
```

ThorVG renders SVG to a pixel buffer. pico-sdl wraps that buffer
as an `SDL_Texture` and draws it via the existing layer system.

### New API

```c
pico_output_draw_svg(path, rect)   // like draw_image
pico_layer_svg(name, path)         // like layer_image
```

### Pros
- **Minimal disruption** -- SVG becomes just another layer type
- Reuses existing layer/hash/TTL infrastructure
- SVG scales to any resolution without pixelation
- Enables vector art, icons, logos in educational projects
- ThorVG's SVG parser is built-in (no libxml dependency)

### Cons
- Adds ~150KB to binary size
- SVG rendering is CPU-intensive (rasterized on load)
- Re-rasterization needed on resize/zoom
- Meson build system (ThorVG) vs Makefile (pico-sdl)
  - can pre-build as static lib, or use pkg-config
- C++ dependency (ThorVG core is C++, linked via C API)

### Verdict: **RECOMMENDED** -- low risk, high value

---

## 4. Replacing SDL2_gfx Primitives with ThorVG

### What SDL2_gfx provides today

| Primitive       | Filled               | Stroke               |
|-----------------|---------------------|-----------------------|
| Triangle        | `filledTrigonRGBA`  | `trigonRGBA`          |
| Ellipse/Circle  | `filledEllipseRGBA` | `ellipseRGBA`         |
| Polygon (N-gon) | `filledPolygonRGBA` | `polygonRGBA`         |

Rects, lines, and points use SDL2 native (NOT SDL2_gfx).

### What ThorVG would provide instead

```c
// Triangle via ThorVG path
Tvg_Paint* shape = tvg_shape_new();
tvg_shape_move_to(shape, x1, y1);
tvg_shape_line_to(shape, x2, y2);
tvg_shape_line_to(shape, x3, y3);
tvg_shape_close(shape);
tvg_shape_set_fill_color(shape, r, g, b, a);
// or stroke:
tvg_shape_set_stroke_width(shape, w);
tvg_shape_set_stroke_color(shape, r, g, b, a);

// Ellipse via ThorVG
tvg_shape_append_circle(shape, cx, cy, rx, ry);

// Rect via ThorVG (with optional rounded corners!)
tvg_shape_append_rect(shape, x, y, w, h, rx, ry);
```

### Pros
- **Drop SDL2_gfx dependency** entirely (-1 dependency)
- **Anti-aliased rendering** -- ThorVG uses sub-pixel AA;
  SDL2_gfx uses aliased integer rasterization
- **Rounded rectangles** -- free with `append_rect(rx, ry)`
- **Gradients** -- linear and radial fills on any shape
- **Stroke features** -- configurable width, cap, join, dash
  (SDL2_gfx stroke is always 1px)
- **Bezier curves** -- `cubic_to()` for smooth paths
- **Consistent quality** -- all primitives rendered by same
  engine with same quality level
- **Path operations** -- arbitrary vector paths, not just
  preset shapes

### Cons
- **Performance model change:**
  SDL2_gfx renders directly to `SDL_Renderer` (GPU path).
  ThorVG SW renders to a CPU buffer, then must be uploaded
  as a texture. This means:
  - Extra CPU work for rasterization
  - Extra GPU upload (`SDL_UpdateTexture`) per frame
  - For simple shapes, SDL2_gfx is likely faster
- **Rendering architecture mismatch:**
  pico-sdl uses `SDL_Renderer` (retained GPU textures).
  ThorVG SW uses CPU buffers. Every ThorVG draw would need:
  1. Render to buffer (CPU)
  2. Upload buffer to SDL_Texture (CPU->GPU)
  3. SDL_RenderCopy to screen (GPU)
  This is a CPU->GPU round-trip per primitive.
- **Immediate mode vs retained mode:**
  pico-sdl draws primitives immediately (fire-and-forget).
  ThorVG builds a scene graph, then renders all at once.
  Adapting pico-sdl's immediate API to ThorVG's retained
  model requires either:
  - (a) Building a scene graph per frame and flushing, or
  - (b) Rendering each primitive individually (wasteful)
- **No direct SDL_Renderer integration:**
  ThorVG doesn't draw to `SDL_Renderer`. It draws to its
  own buffer. This creates a fundamental impedance mismatch.
- **Complexity increase** for what are currently 1-line calls

### Verdict: **NOT RECOMMENDED as full replacement**

The performance penalty and architectural mismatch make a full
replacement impractical. However, ThorVG primitives could be
used **selectively** for features SDL2_gfx can't do (see
section 7).

---

## 5. Effect on Existing Raster-Based Operations

### 5.1 Layers

**No impact.** Layers are `SDL_Texture` objects managed by
`SDL_Renderer`. ThorVG integration would create new layers
(SVG layers) that fit into the existing system. The layer
hash, TTL cleanup, `pico_set_layer()`, `pico_output_draw_layer()`
all work unchanged.

An SVG layer would be created like a buffer layer:
```
ThorVG buffer --> SDL_CreateTexture --> stored as layer
```

### 5.2 Images (PNG/JPG)

**No impact.** `IMG_LoadTexture()` continues to work for raster
images. SVG would be a new format alongside PNG/JPG, not a
replacement. Could optionally route `.svg` files through ThorVG
in `pico_output_draw_image()` (auto-detect by extension).

### 5.3 Buffers (Raw RGBA)

**No impact.** Raw pixel buffers (`Pico_Color_A[]`) go through
`SDL_CreateRGBSurfaceWithFormatFrom()`. This path is independent.

ThorVG's output IS a raw RGBA buffer, so there's a natural
bridge: ThorVG renders to buffer, buffer becomes texture.
This is the same path buffers already use.

### 5.4 Video (Y4M)

**No impact.** Video uses `SDL_PIXELFORMAT_YV12` with streaming
textures and `SDL_UpdateYUVTexture()`. Completely separate path.

### Summary

| Component | Impact    | Reason                           |
|-----------|-----------|----------------------------------|
| Layers    | None      | SVG layers = new layer type      |
| Images    | None      | SVG alongside PNG/JPG            |
| Buffers   | None      | ThorVG output = RGBA buffer      |
| Video     | None      | Separate YUV pipeline            |
| Clear/BG  | None      | SDL_RenderClear unchanged        |
| Push/Pop  | None      | State stack unchanged            |

---

## 6. Effect on Text Drawing

### Current text system
- `SDL_ttf` + embedded `tiny_ttf.h` fallback
- Renders text to `SDL_Surface` then `SDL_Texture`
- Cached as layers via hash/TTL

### ThorVG text capabilities
- Built-in TTF loader (no FreeType, no SDL_ttf)
- `tvg_text_new()`, `tvg_text_set_font()`, `tvg_text_set_text()`
- Supports: font name, size, fill color, gradients
- UTF-8 Unicode support
- Text rendered as vector paths (scalable)

### Could ThorVG replace SDL_ttf?

**Pros:**
- Drop `SDL_ttf` dependency (-1 dependency)
- Text scales without re-rasterization
- Gradient fills on text (not possible with SDL_ttf)
- Vector text = sharp at any zoom level

**Cons:**
- Same CPU buffer -> texture upload overhead
- ThorVG text is newer/less mature than SDL_ttf
- No `TTF_RenderText_Solid` equivalent (retained model)
- Would need to rasterize text to buffer, then upload
- Current caching via layer system works well already

### Verdict: **POSSIBLE BUT LOW PRIORITY**

The current SDL_ttf approach works well and text is already
cached as textures. Replacing it adds complexity for marginal
benefit. However, ThorVG text could be offered as an
**alternative** for vector-quality text when needed.

---

## 7. Rotation, Flip, Scaling for Primitives

### Current situation

**Layers** support all three via `SDL_RenderCopyEx`:
- Rotation: 0-360 degrees, custom anchor point
- Flip: horizontal, vertical, both
- Scale: implicit via `view.dst` / `view.src` / `view.dim`

**Primitives** support NONE of these. SDL2_gfx functions
(`filledTrigonRGBA`, `filledEllipseRGBA`, etc.) take absolute
pixel coordinates and render immediately with no transform
parameters. There is no way to rotate a triangle or flip an
ellipse.

### What ThorVG enables

Every ThorVG paint object supports **per-object transforms**:

```c
// Convenience methods (compose internally):
tvg_paint_translate(shape, tx, ty);
tvg_paint_rotate(shape, degrees);  // clockwise
tvg_paint_scale(shape, factor);

// OR full 3x3 affine matrix:
Tvg_Matrix m = {
    .e11 = cos_a, .e12 = -sin_a, .e13 = tx,
    .e21 = sin_a, .e22 =  cos_a, .e23 = ty,
    .e31 = 0,     .e32 = 0,      .e33 = 1
};
tvg_paint_set_transform(shape, &m);
```

**Flip** = scale with negative factor:
```c
// Horizontal flip around center:
Tvg_Matrix flip_h = {
    -1, 0, width,   // mirror X, translate back
     0, 1, 0,
     0, 0, 1
};
```

**Hierarchical transforms** via Scene containers:
```c
Tvg_Paint* scene = tvg_scene_new();
tvg_paint_translate(scene, 100, 100);
tvg_paint_rotate(scene, 45);
tvg_scene_push(scene, child_shape); // child inherits
```

### How this maps to pico-sdl

If primitives were rendered via ThorVG, they could support
the same transforms as layers. Two approaches:

**Approach A: Per-draw transforms (immediate style)**
```c
pico_set_rotation(45);          // state (push/pop aware)
pico_output_draw_rect(rect);    // draws rotated rect
pico_set_rotation(0);           // reset
```

**Approach B: Layer-based (current architecture)**
Draw primitives into a ThorVG-backed layer, then transform
the layer as usual. This already works -- you can draw into
a layer and rotate the layer. ThorVG just makes the layer
contents higher quality (AA, gradients, etc.).

### Verdict

ThorVG makes per-primitive rotation/flip/scale **possible**.
But it only works for ThorVG-rendered primitives (not SDL2_gfx
ones). This is a strong argument for using ThorVG for ALL
primitive rendering if transforms are desired.

**Trade-off**: transforms on primitives vs. performance.

---

## 8. Other ThorVG Properties (Beyond Transforms)

### 8.1 Per-Object Opacity

```c
tvg_paint_set_opacity(shape, 128);  // 0-255
```
Currently pico-sdl has global `S.alpha` only. ThorVG allows
**per-shape** opacity without changing global state.

### 8.2 Blend Modes (16 modes)

```c
tvg_paint_set_blend_method(shape, TVG_BLEND_METHOD_MULTIPLY);
```
- Normal, Multiply, Screen, Overlay
- Darken, Lighten, ColorDodge, ColorBurn
- HardLight, SoftLight, Difference, Exclusion
- Hue, Saturation, Color, Luminosity, Add

pico-sdl currently uses only `SDL_BLENDMODE_BLEND` (fixed).
ThorVG unlocks compositing effects.

### 8.3 Fill Rules

```c
tvg_shape_set_fill_rule(shape, TVG_FILL_RULE_EVEN_ODD);
tvg_shape_set_fill_rule(shape, TVG_FILL_RULE_NON_ZERO);
```
Controls how overlapping sub-paths are filled. Important
for complex polygons and SVG rendering. Not available in
SDL2_gfx.

### 8.4 Gradient Fills

```c
// Linear gradient
Tvg_Gradient* grad = tvg_linear_gradient_new();
tvg_linear_gradient_set(grad, x1, y1, x2, y2);
Tvg_Color_Stop stops[2] = {
    {0.0, 255, 0, 0, 255},   // red at start
    {1.0, 0, 0, 255, 255}    // blue at end
};
tvg_gradient_set_color_stops(grad, stops, 2);
tvg_shape_set_fill_gradient(shape, grad);

// Radial gradient
Tvg_Gradient* rgrad = tvg_radial_gradient_new();
tvg_radial_gradient_set(rgrad, cx, cy, radius);
```

### 8.5 Stroke Properties

| Property   | SDL2_gfx   | ThorVG                     |
|------------|------------|----------------------------|
| Width      | 1px only   | Any float value            |
| Cap style  | none       | butt, round, square        |
| Join style | none       | miter, round, bevel        |
| Dash       | none       | arbitrary dash patterns    |
| Trim       | none       | partial path rendering     |
| Gradient   | none       | gradient stroke fill       |

```c
tvg_shape_set_stroke_width(shape, 3.0f);
tvg_shape_set_stroke_cap(shape, TVG_STROKE_CAP_ROUND);
tvg_shape_set_stroke_join(shape, TVG_STROKE_JOIN_ROUND);
float dash[] = {10, 5, 3, 5};
tvg_shape_set_stroke_dash(shape, dash, 4, 0);
```

### 8.6 Masking and Clipping

```c
// Clip: show only where clipper shape exists
tvg_paint_set_clip(target, clipper_shape);

// Mask: blend using alpha/luma of mask shape
tvg_paint_set_mask_method(target, mask, TVG_MASK_METHOD_ALPHA);
```
Mask methods: alpha, inverse-alpha, luma, inverse-luma,
add, subtract, intersect, difference.

Currently pico-sdl has only rectangular clipping via
`SDL_RenderSetClipRect`.

### 8.7 Scene Effects

```c
// Gaussian blur
tvg_scene_add_effect_gaussian_blur(scene, 10, 0, 0, 100);

// Drop shadow
tvg_scene_add_effect_drop_shadow(scene,
    0, 0, 0, 128,    // shadow color (RGBA)
    135, 5, 3, 100);  // angle, dist, sigma, quality

// Tint
tvg_scene_add_effect_tint(scene,
    0, 0, 0,          // black point
    255, 200, 100,     // white point
    200);              // intensity
```

### 8.8 Visibility Toggle

```c
tvg_paint_set_visible(shape, false);  // hide without remove
```
Useful for show/hide without rebuilding the scene graph.

### Summary: Property Comparison

| Property        | SDL2/gfx  | ThorVG           |
|-----------------|-----------|------------------|
| Rotation        | Layers    | Any paint        |
| Flip            | Layers    | Any paint        |
| Scale           | Layers    | Any paint        |
| Opacity         | Global    | Per-paint        |
| Blend modes     | 1 mode    | 16 modes         |
| Fill color      | Solid     | Solid + gradient |
| Fill rule       | N/A       | Even-odd / NZ    |
| Stroke width    | 1px       | Any float        |
| Stroke cap      | N/A       | butt/round/sq    |
| Stroke join     | N/A       | miter/round/bvl  |
| Stroke dash     | N/A       | Arbitrary        |
| Clipping        | Rect only | Any shape        |
| Masking         | None      | 10 mask methods  |
| Effects         | None      | Blur/shadow/tint |
| Visibility      | N/A       | Per-paint toggle |
| Anti-aliasing   | None      | Sub-pixel AA     |

---

## 9. How Exactly to Integrate ThorVG with SDL2

### 9.1 Colorspace Match

```
TVG_COLORSPACE_ARGB8888  <-->  SDL_PIXELFORMAT_ARGB8888
```
Byte-for-byte compatible. No conversion needed.

**Alpha caveat**: ThorVG defaults to **premultiplied** alpha.
SDL's `SDL_BLENDMODE_BLEND` expects **straight** alpha.
Two options:
- Use `TVG_COLORSPACE_ARGB8888S` (straight) -- simpler,
  slight performance cost
- Use premultiplied + custom SDL blend mode:
```c
SDL_BlendMode pm_blend = SDL_ComposeCustomBlendMode(
    SDL_BLENDFACTOR_ONE,
    SDL_BLENDFACTOR_ONE_MINUS_SRC_ALPHA,
    SDL_BLENDOPERATION_ADD,
    SDL_BLENDFACTOR_ONE,
    SDL_BLENDFACTOR_ONE_MINUS_SRC_ALPHA,
    SDL_BLENDOPERATION_ADD
);
SDL_SetTextureBlendMode(texture, pm_blend);
```

### 9.2 Integration Pattern for pico-sdl

pico-sdl uses `SDL_Renderer` with `SDL_Texture` targets.
The integration point is: **ThorVG renders to a buffer,
buffer becomes an SDL_Texture.**

```
                     pico-sdl layer system
                     ┌─────────────────────────┐
                     │  SDL_Texture (layer)     │
                     │  - hash key, TTL, view   │
                     └───────────┬─────────────┘
                                 │
         ┌───────────────────────┼─────────────┐
         │                       │             │
    ┌────┴────┐           ┌──────┴───┐   ┌─────┴────┐
    │ SDL2    │           │ SDL2_img │   │ ThorVG   │
    │ native  │           │          │   │ SwCanvas │
    │ rect/   │           │ PNG/JPG  │   │          │
    │ line/pt │           │ loader   │   │ SVG/     │
    └─────────┘           └──────────┘   │ Lottie/  │
                                         │ shapes   │
         SDL2_gfx                        └──────────┘
    ┌─────────┐                              │
    │ tri/    │                    uint32_t[] buffer
    │ oval/   │                              │
    │ poly    │                    SDL_UpdateTexture()
    └─────────┘                              │
                                       SDL_Texture
```

### 9.3 Concrete Code Pattern

#### Initialization (once, in `pico_open`)

```c
#ifdef PICO_THORVG
#include <thorvg_capi.h>

static Tvg_Canvas* tvg_canvas = NULL;
static uint32_t*   tvg_buffer = NULL;
static int         tvg_w = 0, tvg_h = 0;

static void thorvg_init (void) {
    tvg_engine_init(TVG_ENGINE_SW, 0);
}

static void thorvg_term (void) {
    if (tvg_canvas) {
        tvg_canvas_destroy(tvg_canvas);
    }
    free(tvg_buffer);
    tvg_engine_term(TVG_ENGINE_SW);
}
#endif
```

#### SVG Rendering (per-request, cached as layer)

```c
static SDL_Texture* thorvg_render_svg (
    const char* path, int w, int h
) {
    // Resize buffer if needed
    if (w != tvg_w || h != tvg_h) {
        free(tvg_buffer);
        tvg_buffer = calloc(w * h, sizeof(uint32_t));
        tvg_w = w;
        tvg_h = h;
    } else {
        memset(tvg_buffer, 0, w * h * sizeof(uint32_t));
    }

    // Create or reconfigure canvas
    if (tvg_canvas) {
        tvg_canvas_destroy(tvg_canvas);
    }
    tvg_canvas = tvg_swcanvas_create();
    tvg_swcanvas_set_target(
        tvg_canvas, tvg_buffer, w, w, h,
        TVG_COLORSPACE_ARGB8888S
    );

    // Load SVG
    Tvg_Paint* pic = tvg_picture_new();
    tvg_picture_load(pic, path);
    tvg_picture_set_size(pic, w, h);

    // Render
    tvg_canvas_push(tvg_canvas, pic);
    tvg_canvas_draw(tvg_canvas, 1);
    tvg_canvas_sync(tvg_canvas);

    // Create SDL_Texture from buffer
    SDL_Texture* tex = SDL_CreateTexture(
        G.ren,
        SDL_PIXELFORMAT_ARGB8888,
        SDL_TEXTUREACCESS_STATIC,
        w, h
    );
    SDL_UpdateTexture(
        tex, NULL, tvg_buffer,
        w * sizeof(uint32_t)
    );
    SDL_SetTextureBlendMode(tex, SDL_BLENDMODE_BLEND);

    return tex;
}
```

#### Usage in pico_output_draw_image (auto-detect SVG)

```c
void pico_output_draw_image (const char* path, ...) {
#ifdef PICO_THORVG
    const char* ext = strrchr(path, '.');
    if (ext && strcasecmp(ext, ".svg") == 0) {
        // route to ThorVG
        SDL_Texture* tex = thorvg_render_svg(path, w, h);
        // ... store as layer, draw via SDL_RenderCopy ...
        return;
    }
#endif
    // existing PNG/JPG path via SDL2_image
    IMG_LoadTexture(G.ren, path);
    ...
}
```

### 9.4 Build Integration (Makefile)

```makefile
# Optional ThorVG support
ifdef THORVG
    CFLAGS  += -DPICO_THORVG $(shell pkg-config --cflags thorvg)
    LDFLAGS += $(shell pkg-config --libs thorvg)
    # Or if using static lib:
    # CFLAGS  += -DPICO_THORVG -I./thorvg/inc
    # LDFLAGS += -L./thorvg/lib -lthorvg -lstdc++
endif
```

### 9.5 Canvas Reuse Strategy

For repeated rendering (animation loop), reuse the canvas:

```c
// Add paints, keep references
tvg_canvas_push(canvas, shape);

// Each frame: modify in place, re-render
tvg_paint_translate(shape, new_x, new_y);
tvg_canvas_update(canvas);      // recalc dirty regions
tvg_canvas_draw(canvas, true);  // rasterize (clear buf)
tvg_canvas_sync(canvas);        // block until done

// Upload to SDL
SDL_UpdateTexture(tex, NULL, buffer, stride);
```

For one-shot rendering (SVG loaded once, cached as texture):
- Create canvas, render, destroy canvas
- Keep only the SDL_Texture

### 9.6 Thread Safety Rules

- All ThorVG calls on **one thread** (main thread)
- Internal parallelism via `tvg_engine_init(n_threads)`
- Do NOT read buffer between `draw()` and `sync()`
- Paint objects cannot be shared across canvases

---

## 10. What is NOT Supported That COULD Be Supported

With ThorVG, pico-sdl could gain these **new capabilities**:

### 7.1 SVG Rendering (HIGH VALUE)
- Load and display SVG files as scalable images
- Vector icons, logos, UI elements
- Resolution-independent graphics
- `pico_output_draw_svg(path, rect)`

### 7.2 Lottie Animations (HIGH VALUE)
- Load and play Lottie JSON animations
- Frame-based control (like video but vector)
- Lightweight animated graphics
- `pico_output_draw_lottie(path, rect)`
- `pico_set_lottie(name, frame)`

### 7.3 Rounded Rectangles (MEDIUM VALUE)
- `tvg_shape_append_rect(x, y, w, h, rx, ry)`
- Currently impossible with SDL2_gfx
- Very common UI element
- Could extend: `pico_output_draw_rect()` with radius param

### 7.4 Gradient Fills (MEDIUM VALUE)
- Linear and radial gradients on any shape
- Color stops with arbitrary positions
- Not possible with current SDL2_gfx
- `pico_set_gradient_linear(x1,y1, x2,y2, stops)`
- `pico_set_gradient_radial(cx,cy, r, stops)`

### 7.5 Anti-Aliased Primitives (MEDIUM VALUE)
- ThorVG uses sub-pixel anti-aliasing
- SDL2_gfx primitives are aliased (jagged edges)
- Could offer AA versions of tri, oval, poly

### 7.6 Bezier Curves / Paths (MEDIUM VALUE)
- Arbitrary vector paths with cubic beziers
- `pico_output_draw_path(points, types)`
- Not possible with current primitives

### 7.7 Stroke Customization (LOW-MEDIUM VALUE)
- Variable stroke width (currently always 1px)
- Cap styles: butt, round, square
- Join styles: miter, round, bevel
- Dash patterns
- `pico_set_stroke_width(w)`
- `pico_set_stroke_cap(cap)`

### 7.8 Masking and Clipping (LOW VALUE)
- Arbitrary shape masks
- Path-based clipping regions
- Beyond current `SDL_RenderSetClipRect`

### 7.9 Effects (LOW VALUE)
- Blur, drop shadow, tint
- Applied to any paint object
- Heavy for educational use

### 7.10 TVG Binary Format (LOW VALUE)
- Pre-compiled vector graphics
- Faster loading than SVG
- Smaller file size (~30% less than SVG)
- Asset pipeline optimization

---

## 11. Recommended Integration Strategy

### Phase 1: SVG Support (Additive, Low Risk)
- Add ThorVG as optional dependency
- Implement `pico_output_draw_svg()` / `pico_layer_svg()`
- SVG rendered to buffer -> SDL_Texture -> layer
- No changes to existing code paths
- Build: `make THORVG=1` to enable

### Phase 2: Lottie Animations (Additive, Low Risk)
- Implement `pico_output_draw_lottie()`
- Frame control similar to video API
- Extends layer system with animation type

### Phase 3: Enhanced Primitives (Selective, Medium Risk)
- Add NEW functions (don't replace existing):
  - `pico_output_draw_rrect()` -- rounded rect
  - `pico_output_draw_path()` -- bezier paths
  - `pico_set_gradient_*()` -- gradient fills
  - `pico_set_stroke_width()` -- variable stroke
- Keep SDL2_gfx for simple cases (performance)
- Use ThorVG only for features SDL2_gfx can't do

### NOT Recommended
- Full replacement of SDL2_gfx (performance loss)
- Full replacement of SDL_ttf (unnecessary complexity)
- ThorVG GL backend (conflicts with SDL_Renderer)

---

## 12. Build Integration Options

### Option A: Pre-built Static Library
```makefile
# Build ThorVG once:
# meson setup build -Ddefault_library=static \
#   -Dbindings=capi -Dloaders=svg,lottie,png \
#   -Dengines=sw
# ninja -C build

THORVG_CFLAGS = -I/path/to/thorvg/inc
THORVG_LIBS = -L/path/to/thorvg/lib -lthorvg -lstdc++
```

### Option B: System Package
```makefile
THORVG_CFLAGS = $(shell pkg-config --cflags thorvg)
THORVG_LIBS = $(shell pkg-config --libs thorvg)
```

### Option C: Git Submodule
```
git submodule add https://github.com/thorvg/thorvg
# Build as part of pico-sdl's Makefile
```

Note: ThorVG is C++ so linking requires `-lstdc++` even
when using the C API.

---

## 13. Summary Table

| Feature                | Effort | Value  | Risk | Recommend?     |
|------------------------|--------|--------|------|----------------|
| SVG rendering          | Low    | High   | Low  | YES            |
| Lottie animations      | Low    | High   | Low  | YES            |
| Rounded rectangles     | Low    | Medium | Low  | YES            |
| Gradient fills         | Medium | Medium | Low  | YES            |
| Anti-aliased prims     | Medium | Medium | Med  | SELECTIVE      |
| Bezier paths           | Medium | Medium | Low  | YES            |
| Stroke customization   | Low    | Low    | Low  | YES            |
| Primitive transforms   | Medium | High   | Med  | YES (via TVG)  |
| Blend modes (16)       | Low    | Medium | Low  | YES            |
| Shape masking/clipping | Medium | Low    | Med  | LATER          |
| Scene effects          | Medium | Low    | Med  | LATER          |
| Replace SDL2_gfx       | High   | Low    | High | NO             |
| Replace SDL_ttf        | High   | Low    | Med  | NO             |
| ThorVG GL backend      | High   | Low    | High | NO             |

---

## Pending Actions

- [ ] User decision: which phases to pursue
- [ ] User decision: build integration approach (A/B/C)
- [ ] Prototype SVG rendering path
- [ ] Test ThorVG buffer -> SDL_Texture pipeline
- [ ] Benchmark: ThorVG primitives vs SDL2_gfx
- [ ] Design API extensions for new primitives
