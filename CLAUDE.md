# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

"Grid Noise" — a single-file, zero-dependency generative art tool. Layers of noise are quantized to a pixel grid and rendered as colored squares / circles / crosses / stripes on a dark background, animated on the GPU. The whole app is `index.html` (CSS + HTML + JS + GLSL inline). There is no build step, no package.json, no tests, no lint.

## Running it

Opening `index.html` via `file://` is unreliable (WebGL2 / `captureStream` restrictions). Serve it over HTTP:

```bash
python3 -m http.server 8080      # then open http://localhost:8080
```

To verify a change renders without a real browser, headless Chrome works (software GL):

```bash
google-chrome --headless=new --no-sandbox --use-gl=angle --use-angle=swiftshader \
  --enable-unsafe-swiftshader --window-size=1400,900 --virtual-time-budget=3000 \
  --screenshot=shot.png http://localhost:8080/index.html
```

The badge in the canvas's top-right corner reads `WebGL2` or `Canvas 2D (fallback)` — if a shader change breaks compilation the app silently falls back to Canvas 2D, so always check the badge (or the console) after touching GLSL.

Syntax-check the inline JS without a browser:

```bash
node -e "const h=require('fs').readFileSync('index.html','utf8');new Function(h.match(/<script>([\s\S]*?)<\/script>/)[1]);console.log('ok')"
```

## Architecture

### Render pipeline (WebGL2, `class WebGLRenderer`)

Per frame, `renderer.render(state)`:

1. Clear `accumFBO[0]` to the background color.
2. For each visible layer, in array order (index 0 = bottom):
   - `renderLayer()` draws a fullscreen quad with `LAYER_FRAG_SRC` into that layer's own FBO (one FBO per `layer.id`, cached in `layerFBOs`). The fragment shader does **everything** per pixel: noise → grid quantize (`floor(pixelPos / u_gridSize)`) → threshold → color lookup → SDF shape test on `fract(pixelPos / u_gridSize)`. Output is straight (non-premultiplied) RGBA with alpha = shape coverage.
   - A blend pass with `BLEND_FRAG_SRC` composites that layer texture onto the accumulator using ping-pong between `accumFBO[0]` and `[1]`. Blend modes (normal/multiply/screen/overlay/add) are implemented in GLSL, not via `gl.blendFunc`.
3. `BLIT_FRAG_SRC` copies the final accumulator to the default framebuffer.

All three programs share `VERT_SRC`, which declares `layout(location = 0) in vec2 a_pos` so a single VAO works for every program. Keep that qualifier if you add a program.

Every layer re-renders every frame regardless of whether anything changed. The `_dirty` flag on layers is unused; it's a hook for a future "only re-render changed layers" optimization, not current behavior.

### Enum ordering is load-bearing

The shader receives `u_noiseType`, `u_shapeType`, and `u_blendMode` as integers derived from `indexOf()` on these JS arrays:

```js
NOISE_TYPES    = ['perlin','simplex','worley','fbm','white','sine','value']   // 0..6
SHAPES         = ['square','circle','cross','vstripes','hstripes','mixed']   // 0..5
BLEND_MODES_GL = ['normal','multiply','screen','overlay','add']              // 0..4
```

The `if (u_noiseType == N)` / `getShape()` / blend branches in the GLSL and the `BLEND_MODES_2D` fallback array are all positional. Adding or reordering an entry means updating the JS array, the GLSL dispatch, and the Canvas 2D fallback together.

### Two renderers must stay in sync

`renderCanvas2D()` plus the `_perlin2`/`_worley2`/… JS functions are a full CPU port used only when WebGL2 is unavailable or a shader fails to compile. Any change to how a layer looks (new shape, new noise, threshold semantics, global scale, padding) needs the equivalent change in both the GLSL and the JS fallback. The fallback also replaces the `<canvas>` element on activation because a canvas that already has a WebGL context cannot hand out a 2D context — hence `canvas` is `let`, not `const`.

### State model

`state` is a plain object; `state.layers` is the ordered layer array, each from `makeLayer()`. There is no reactivity — sliders mutate layer fields directly and the next `requestAnimationFrame` picks them up as uniforms. `buildLayerUI()` tears down and rebuilds the entire layer panel; call it after structural changes (add/remove/reorder/color count), not on slider drags.

`state.globalScale` multiplies every layer's `gridSize`; the effective cell size is `Math.round(gridSize * globalScale)` in both renderers so cells never land on fractional pixels.

Animation is `state.time` advancing by `dt * masterSpeed` when `state.playing`; the shader offsets noise coordinates by `u_time * u_animSpeed * 0.01`. A layer with `animSpeed: 0` is static even while playing.

### UI gotchas

- Only `.layer-header` is `draggable`, not the whole card. Making the card draggable hijacks range-input drags inside it.
- Color swatches: left-click removes, right-click opens a picker. `<input type="color">` is created on the fly and `.click()`ed rather than kept in the DOM.
- `exportGIF()` actually records WebM via `MediaRecorder` (4 s); `exportPNG()` uses `gl.readPixels` and flips Y manually because WebGL's origin is bottom-left.

## GLSL pitfalls hit in this project

- `half` is a reserved word in GLSL ES — don't use it as an identifier (it broke shader compilation once; the cross SDF now uses `ext`).
- `uniform vec3 u_colors[8]` is set with one `uniform3fv` call on a 24-float buffer; `u_colorCount` caps the lookup. Max 8 colors per layer is enforced on both sides.

## Git

Work on `claude/<feature>-<sessionid>` branches and push with `git push -u origin <branch>`. The repo currently has a single branch, which is also the default; Vercel deploys from it.
