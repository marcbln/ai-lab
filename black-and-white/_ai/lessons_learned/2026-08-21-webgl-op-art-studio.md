# [2026-08-21] - Black & White WebGL Op-Art Studio

## Context
Built `gpt-5.6-luna/index.html`: a self-contained single-file WebGL app with 7 raymarched
signed-distance-field (SDF) effects, strict black-and-white op-art rendering (posterized
contour bands), mouse orbit/zoom camera, lil-gui parameter panel, and keyboard shortcuts.
Verified end-to-end with headless Chromium + Playwright.

## Challenge
1. The page appeared **completely broken** (rendered pure white, 99.6% of pixels) when
   screenshotted via the `chromium --headless=new --screenshot` CLI — even though a trivial
   three.js test scene worked fine.
2. One effect ("Fly Tunnel") rendered all-black while every other mode worked.
3. Debugging WebGL shaders headlessly is hard: no console visibility, no easy pixel reads.

## Discovery/Solution

### Headless Chrome CLI screenshots lie for WebGL
- `chromium --headless=new --screenshot` used a **deprecated automatic software-GL fallback**
  that produced garbage (all-white canvas) for the raymarched scene.
- The fix: drive the page with **Playwright launched with `--enable-unsafe-swiftshader`**.
  Same page rendered correctly (black sky, white terrain bands). Lesson: never judge a WebGL
  page's correctness from a raw headless-CLI screenshot — pixel statistics are only trustworthy
  from a properly configured SwiftShader context.

### `canvas.getContext()` returns null with mismatched attributes
- three.js creates the context with specific attributes (`antialias: true`, etc.). Calling
  `canvas.getContext('webgl')` again with default attributes returns **null** per spec.
- Fix: expose the live context from the page (`window.__gl = renderer.getContext()`) and read
  pixels through it, instead of re-requesting the context.

### Raymarching: "camera starts inside the geometry" bug
- The original Fly Tunnel put the camera on the tube axis, *inside* the SDF. `march()` returns
  `t ≈ 0` immediately, and the standard `if (t > 0.0)` hit test treats that as a miss → the
  whole screen renders black (not an error, just no hit).
- Robust fix chosen: design scenes so the camera is **outside** the SDF (oblique 3/4 view of a
  ribbed, scrolling ring tunnel instead of an in-axis fly-through). Alternative (not taken):
  handle interior marching by stepping forward through the interior to the far wall.

### Fast shader debugging via output-line swapping
- Bisect shader bugs by replacing only the final `gl_FragColor` line with diagnostics:
  1. hit test: `vec3(t > 0.0 ? 1.0 : 0.0)` → does the raymarch hit?
  2. raw shading: `vec3(clamp(shade, 0.0, 1.0))` → is the lighting sane?
  3. exact values via `gl.readPixels` grid samples through the exposed context.
- Keep the diagnostic scope tight (GLSL scoping: `shade` declared inside an `if` block is not
  visible at the final line — a diagnostic patch that referenced it broke compilation).

### Verification workflow for effect studios
- Exercise **every mode** (keys 1-7 AND the GUI), then check pixel distributions per mode —
  each effect should show a meaningful black/white split, not ~0% or ~100% white.
- Capture `pageerror` + `console.error` via Playwright listeners; absence of errors is not
  proof — confirm each mode actually renders.
- lil-gui's Effect dropdown is a **native `<select>`** — test it with `select_option`, not by
  clicking `.lil-gui .list li` (those elements don't exist in lil-gui 0.18).

## Key Takeaways
- Use **Playwright + `--enable-unsafe-swiftshader`** (never the bare headless CLI) to verify
  WebGL pages; treat CLI screenshot pixel stats as untrustworthy.
- For pixel probing, expose `renderer.getContext()` and reuse it — re-`getContext` fails on
  attribute mismatch.
- When a raymarched scene renders fully black, suspect "camera inside the SDF" before anything
  else; prefer camera-outside scene designs for simplicity.
- Shader bugs: swap only the `gl_FragColor` line for staged diagnostics, mind GLSL scoping.
- Verify all switchable modes + interactions (drag orbit, wheel zoom, GUI, keys) and require a
  sensible B&W pixel distribution per mode, not just "no console errors".
