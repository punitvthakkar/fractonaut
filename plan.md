# Fractonaut Fix Plan

Implementation plan for every confirmed issue in `AUDIT.md`, **except perturbation-theory rendering, which is explicitly excluded by the project owner — do not implement it, do not add any library for it.**

Written against commit `a17cadc`. Every "Anchor" below is an exact string that exists in the current source — **locate code by searching for the anchor string, never by line number** (line numbers shift as you edit).

---

## Ground rules for the implementing model

1. **No frameworks, no build step, no npm.** This project is plain JS/GLSL/CSS served as static files. Vendored files (fonts, one .mjs) are committed directly to the repo.
2. **Match existing style:** 4-space indentation, single quotes, existing comment tone. Do not reformat code you are not changing.
3. **After every edit to `script.js` or `sw.js`, run** `node --check script.js` / `node --check sw.js` (for the `.mjs` file: `node --check mp4-muxer.mjs` is not needed — don't touch its contents). Fix syntax errors before moving on.
4. **One git commit per task**, using the commit message given. Work on a branch: `git checkout -b fix/audit-2026-06`.
5. To test in a browser, serve over HTTP (service worker does not run from `file://`): `python3 -m http.server 8000` then open `http://localhost:8000`.
6. If a task's anchor string cannot be found, STOP that task and report it — do not guess at a similar-looking location.

## Suggested subagent split

The work is two independent streams plus a final verification pass:

- **Subagent 1 — Stream A** (owns `index.html`, `style.css`, `sw.js`, `manifest.json`, new asset files): tasks A1–A5.
- **Subagent 2 — Stream B** (owns `script.js` exclusively): tasks B1–B14, **strictly in order** (later tasks call functions introduced by earlier ones).
- Streams A and B can run in parallel — they touch disjoint files. The only cross-stream dependency: B11 changes an import path to a file A1 downloads; B11's *edit* can be made regardless, but video export only *works* once A1 is done.
- **Subagent 3 — Verifier**: after both streams finish, run the Acceptance checklist at the bottom and the grep audits, and report failures.

If running as a single agent, do Stream B first, then Stream A, then verify.

---

# Stream A — assets, HTML, CSS, service worker

## A1. Vendor mp4-muxer (audit #7)

**Goal:** remove the runtime unpkg.com dependency.

```bash
cd <repo root>
curl -sL "https://unpkg.com/mp4-muxer@5.2.2/build/mp4-muxer.mjs" -o mp4-muxer.mjs
head -c 300 mp4-muxer.mjs   # sanity: should be JS, not an HTML error page
```

Keep the file's own license header intact (it is MIT). Commit the file.
The `script.js` import-path change is task **B11** (Stream B owns that file).

**Commit:** `Vendor mp4-muxer 5.2.2 locally (remove runtime unpkg dependency)`

## A2. Self-host fonts, remove Google Fonts (audit #8)

**Goal:** no external font requests; `Space Mono` (referenced 5× in style.css but never loaded) actually loads.

1. Download the latin woff2 files:

```bash
mkdir -p fonts
UA="Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0 Safari/537.36"
curl -sA "$UA" "https://fonts.googleapis.com/css2?family=Outfit:wght@300;400;600;700&display=swap" -o /tmp/outfit.css
curl -sA "$UA" "https://fonts.googleapis.com/css2?family=Space+Mono&display=swap" -o /tmp/spacemono.css
```

2. In each CSS file, find the `@font-face` block whose preceding comment is `/* latin */` for each weight (300/400/600/700 for Outfit; 400 for Space Mono). Copy the `src: url(...)` URL from each latin block and download:

```bash
curl -s "<latin-300-url>" -o fonts/outfit-300.woff2
curl -s "<latin-400-url>" -o fonts/outfit-400.woff2
curl -s "<latin-600-url>" -o fonts/outfit-600.woff2
curl -s "<latin-700-url>" -o fonts/outfit-700.woff2
curl -s "<spacemono-latin-400-url>" -o fonts/space-mono-400.woff2
file fonts/*.woff2   # all should say "Web Open Font Format"
```

3. At the very top of `style.css`, add:

```css
@font-face { font-family: 'Outfit'; font-style: normal; font-weight: 300; font-display: swap; src: url('fonts/outfit-300.woff2') format('woff2'); }
@font-face { font-family: 'Outfit'; font-style: normal; font-weight: 400; font-display: swap; src: url('fonts/outfit-400.woff2') format('woff2'); }
@font-face { font-family: 'Outfit'; font-style: normal; font-weight: 600; font-display: swap; src: url('fonts/outfit-600.woff2') format('woff2'); }
@font-face { font-family: 'Outfit'; font-style: normal; font-weight: 700; font-display: swap; src: url('fonts/outfit-700.woff2') format('woff2'); }
@font-face { font-family: 'Space Mono'; font-style: normal; font-weight: 400; font-display: swap; src: url('fonts/space-mono-400.woff2') format('woff2'); }
```

4. In `index.html`, delete these three lines (anchor: `fonts.googleapis.com`):

```html
    <link rel="preconnect" href="https://fonts.googleapis.com">
    <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
    <link href="https://fonts.googleapis.com/css2?family=Outfit:wght@300;400;600;700&display=swap" rel="stylesheet">
```

**Fallback if fonts.googleapis.com is unreachable from your environment:** still delete the three `<link>` tags, skip the @font-face additions, and report that font files must be added manually.

**Verify:** load the page; DevTools Network tab shows zero requests to any domain other than localhost; headings render in Outfit; the stats readout renders in Space Mono.

**Commit:** `Self-host Outfit and Space Mono fonts (remove Google Fonts dependency)`

## A3. Rewrite sw.js (audit #14, runtime-cache growth, icon precache)

**Goal:** instant cached startup (stale-while-revalidate) with fresh HTML on each visit, precache that includes the new local assets, drop ~1.2 MB of icons from precache, remove the unused `CACHE_VERSION`.

Replace the **entire contents** of `sw.js` with:

```js
// Service Worker for Fractonaut PWA
// IMPORTANT: bump the CACHE version string on EVERY deploy that changes any file below.
const CACHE = 'fractonaut-v2';

const PRECACHE_FILES = [
  './',
  './index.html',
  './script.js',
  './style.css',
  './manifest.json',
  './256.png',
  './mp4-muxer.mjs',
  './fonts/outfit-300.woff2',
  './fonts/outfit-400.woff2',
  './fonts/outfit-600.woff2',
  './fonts/outfit-700.woff2',
  './fonts/space-mono-400.woff2'
];

self.addEventListener('install', (event) => {
  event.waitUntil(caches.open(CACHE).then((cache) => cache.addAll(PRECACHE_FILES)));
  self.skipWaiting();
});

self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys()
      .then((names) => Promise.all(
        names.filter((n) => n !== CACHE).map((n) => caches.delete(n))
      ))
      .then(() => self.clients.claim())
  );
});

self.addEventListener('fetch', (event) => {
  const url = new URL(event.request.url);
  // Same-origin GET only; everything else goes straight to the network
  if (event.request.method !== 'GET' || url.origin !== self.location.origin) {
    return;
  }

  // Network-first for the page itself, so deploys land on the next load
  if (event.request.mode === 'navigate') {
    event.respondWith(
      fetch(event.request)
        .then((response) => {
          if (response && response.status === 200) {
            const copy = response.clone();
            caches.open(CACHE).then((cache) => cache.put('./index.html', copy));
          }
          return response;
        })
        .catch(() => caches.match('./index.html'))
    );
    return;
  }

  // Stale-while-revalidate for static assets: serve cache instantly, refresh in background
  event.respondWith(
    caches.match(event.request, { ignoreSearch: true }).then((cached) => {
      const network = fetch(event.request)
        .then((response) => {
          if (response && response.status === 200) {
            const copy = response.clone();
            caches.open(CACHE).then((cache) => cache.put(event.request, copy));
          }
          return response;
        })
        .catch(() => cached);
      return cached || network;
    })
  );
});
```

Notes baked into this design (do not "improve" them away):
- `ignoreSearch: true` makes the existing `style.css?v=2` reference hit the precached copy.
- Cross-origin requests are not intercepted or cached, so the runtime cache cannot grow unboundedly — after A1/A2 there are no cross-origin requests anyway.
- `512.png`/`1024.png` stay in the repo (used by A4) but are no longer precached.

Also in `manifest.json`: add the 512 icon (PWA installability prefers ≥512). Replace the `icons` array with:

```json
  "icons": [
    { "src": "256.png", "sizes": "256x256", "type": "image/png" },
    { "src": "512.png", "sizes": "512x512", "type": "image/png" }
  ]
```

**Verify:** load page online; DevTools → Application → Service Workers shows the new SW active; go Offline in DevTools and reload — app loads, fonts render. Cache Storage shows only `fractonaut-v2`.

**Commit:** `Rewrite service worker: SWR assets, network-first HTML, lean precache`

## A4. Open Graph / Twitter link previews (audit #4)

In `index.html`, directly after the line `<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">`, insert:

```html
    <!-- Link previews -->
    <link rel="canonical" href="https://fractonaut.com/">
    <meta property="og:type" content="website">
    <meta property="og:site_name" content="Fractonaut">
    <meta property="og:title" content="Fractonaut — explore infinite fractals">
    <meta property="og:description" content="Take a trip through the Mandelbrot set, Julia set and Sierpinski triangle. Zoom into infinite detail and share your favorite locations with friends.">
    <meta property="og:url" content="https://fractonaut.com/">
    <meta property="og:image" content="https://fractonaut.com/1024.png">
    <meta property="og:image:width" content="1024">
    <meta property="og:image:height" content="1024">
    <meta name="twitter:card" content="summary_large_image">
    <meta name="twitter:title" content="Fractonaut — explore infinite fractals">
    <meta name="twitter:description" content="Zoom into infinite fractal detail and share your favorite locations with friends.">
    <meta name="twitter:image" content="https://fractonaut.com/1024.png">
```

**Verify:** `curl -s http://localhost:8000 | grep og:image` returns the tag.

**Commit:** `Add Open Graph and Twitter Card meta tags for link previews`

## A5. Small UI support changes for Stream B

1. **Disabled export options** (supports B13). Add to `style.css` (near other `.resolution-option` rules — anchor: `.resolution-option`):

```css
.resolution-option:disabled,
.resolution-option.unsupported {
    opacity: 0.35;
    cursor: not-allowed;
}
```

2. **Cancel-button label** (supports B9 — declining a shared trip now jumps to the location instead of discarding it). In `index.html`, find the button with `id="cancelAddLocationBtn"` and change its visible label text to `Jump there instantly`. Do not change the id.

**Commit:** `UI support: unsupported-resolution styling, shared-trip cancel label`

---

# Stream B — script.js (strict order B1 → B14)

## B1. Shared URL builder + HTML escaper, full-precision share links (audit #1, part of #13)

**Step 1.** Directly above the line `function getShareText(locationName, shareUrl) {`, insert:

```js
function escapeHtml(s) {
    return String(s).replace(/[&<>"']/g, (c) => (
        { '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;' }[c]
    ));
}

function buildLocationParams(loc) {
    // String(number) is the shortest exact round-trip encoding — never toFixed,
    // which destroys deep-zoom locations (viewport width is 3/zoom).
    return new URLSearchParams({
        x: String(loc.x),
        y: String(loc.y),
        z: String(loc.zoom || (3.0 / state.zoomSize)),
        i: loc.iterations ?? state.maxIterations,
        p: loc.paletteId ?? state.paletteId,
        f: loc.fractalType ?? state.fractalType,
        n: loc.title,
        d: loc.duration || 30
    });
}
```

**Step 2.** In `shareLocation(loc)` — anchor: `const zoom = loc.zoom || (3.0 / state.zoomSize);` followed by `const params = new URLSearchParams({` — replace from the `const zoom =` line through the closing `});` of that URLSearchParams call with:

```js
    const params = buildLocationParams(loc);
```

**Step 3.** In `showShareButtonPopup(loc)` there is a second identical block (same two anchors, second occurrence). Replace it the same way.

**Step 4.** In `saveScene(name, duration)` — anchor: `// 1. Generate URL` — replace from `const zoom = 3.0 / state.zoomSize;` through the closing `});` of its URLSearchParams call with:

```js
    const zoom = 3.0 / state.zoomSize;
    const params = buildLocationParams({
        x: state.zoomCenter.x,
        y: state.zoomCenter.y,
        zoom: zoom,
        iterations: state.maxIterations,
        paletteId: state.paletteId,
        fractalType: state.fractalType,
        title: name,
        duration: duration
    });
```

**Verify:** `node --check script.js`; then `grep -n "toFixed(6)" script.js` — the only remaining hits must be inside `updateStats` (display-only) and the disabled perf-logging block (deleted in B14).

**Commit:** `Encode share URLs at full precision via shared buildLocationParams helper`

## B2. Render-on-demand + full-resolution idle frames (audit #9, #10) — the core rendering task

**Step 1.** Anchor: `let lastTime = 0;` (it sits directly above `// Rendering - Optimized for smoothness`). Replace that single line with:

```js
let lastTime = 0;
let needsRender = true;

function requestRender() {
    needsRender = true;
}
```

**Step 2.** Replace the **entire** `drawScene` function (from `function drawScene(timestamp) {` to its closing `}` — it currently ends with the `mainRenderRAF = requestAnimationFrame(drawScene);` block) with:

```js
function drawScene(timestamp) {
    if (!lastTime) lastTime = timestamp;
    const deltaTime = (timestamp - lastTime) / 1000;
    lastTime = timestamp;

    // Velocity physics - only when not dragging or animating
    if (!state.isDragging && !state.isAnimating) {
        const dt60 = deltaTime * 60;
        state.targetZoomCenter.x -= state.velocity.x * dt60;
        state.targetZoomCenter.y -= state.velocity.y * dt60;

        const frictionPow = Math.pow(state.friction, dt60);
        state.velocity.x *= frictionPow;
        state.velocity.y *= frictionPow;

        // Early zero-out for better performance
        if (Math.abs(state.velocity.x) < 1e-9 && Math.abs(state.velocity.y) < 1e-9) {
            state.velocity.x = 0;
            state.velocity.y = 0;
        }
    }

    // Handle Fidget Zoom
    if (state.fidgetZoomVelocity !== 0) {
        handleZoom(state.fidgetZoomVelocity);
    }

    // Smooth interpolation
    if (!state.isAnimating) {
        const lerpFactor = 1.0 - Math.pow(0.1, deltaTime * 10);

        // Apply smooth zoom limit at 0.5x (zoomSize = 6.0)
        const maxZoomSize = 6.0;
        if (state.targetZoomSize > maxZoomSize) {
            const excess = state.targetZoomSize - maxZoomSize;
            const resistance = 1.0 / (1.0 + excess * 0.5);
            state.targetZoomSize = maxZoomSize + excess * resistance;
            state.targetZoomCenter.x = 0.75;
            state.targetZoomCenter.y = 0.0;
        }

        // Single lerp calculation for all axes
        const diffSize = state.targetZoomSize - state.zoomSize;
        const diffX = state.targetZoomCenter.x - state.zoomCenter.x;
        const diffY = state.targetZoomCenter.y - state.zoomCenter.y;

        state.zoomSize += diffSize * lerpFactor;
        state.zoomCenter.x += diffX * lerpFactor;
        state.zoomCenter.y += diffY * lerpFactor;
    } else {
        state.zoomSize = state.targetZoomSize;
    }

    // Idle detection: once fully settled, snap exactly and stop re-rendering
    const eps = Math.abs(state.zoomSize) * 1e-6;
    const settled = !state.isAnimating && !state.isDragging
        && state.fidgetZoomVelocity === 0
        && state.velocity.x === 0 && state.velocity.y === 0
        && Math.abs(state.targetZoomSize - state.zoomSize) < eps
        && Math.abs(state.targetZoomCenter.x - state.zoomCenter.x) < eps
        && Math.abs(state.targetZoomCenter.y - state.zoomCenter.y) < eps;

    if (settled) {
        state.zoomSize = state.targetZoomSize;
        state.zoomCenter.x = state.targetZoomCenter.x;
        state.zoomCenter.y = state.targetZoomCenter.y;
    }

    // Interaction renders at capped DPR; the settled frame refines at full DPR
    const resized = resizeCanvasToDisplaySize(gl.canvas, settled);

    if (settled && !needsRender && !resized) {
        if (!isExporting) {
            mainRenderRAF = requestAnimationFrame(drawScene);
        }
        return;
    }
    needsRender = false;

    // FPS Calculation (rendered frames only)
    state.frameCount++;
    if (timestamp - state.lastFpsTime >= 500) {
        state.currentFps = Math.round((state.frameCount * 1000) / (timestamp - state.lastFpsTime));
        state.frameCount = 0;
        state.lastFpsTime = timestamp;
    }

    gl.viewport(0, 0, gl.canvas.width, gl.canvas.height);

    gl.clearColor(0.0, 0.0, 0.0, 1.0);
    gl.clear(gl.COLOR_BUFFER_BIT);

    gl.useProgram(programInfo.program);

    // Bind VAO (vertex state already configured at init)
    gl.bindVertexArray(vao);

    // Set uniforms
    gl.uniform2f(programInfo.uniformLocations.resolution, gl.canvas.width, gl.canvas.height);

    const centerXSplit = splitDouble(state.zoomCenter.x);
    const centerYSplit = splitDouble(state.zoomCenter.y);
    const zoomSizeSplit = splitDouble(state.zoomSize);

    gl.uniform2f(programInfo.uniformLocations.zoomCenterX, centerXSplit[0], centerXSplit[1]);
    gl.uniform2f(programInfo.uniformLocations.zoomCenterY, centerYSplit[0], centerYSplit[1]);
    gl.uniform2f(programInfo.uniformLocations.zoomSize, zoomSizeSplit[0], zoomSizeSplit[1]);

    gl.uniform1i(programInfo.uniformLocations.maxIterations, state.maxIterations);
    gl.uniform1i(programInfo.uniformLocations.paletteId, state.paletteId);
    gl.uniform1i(programInfo.uniformLocations.fractalType, state.fractalType);
    gl.uniform2f(programInfo.uniformLocations.juliaC, state.juliaC.x, state.juliaC.y);

    const highPrecision = state.zoomSize < 0.001 && state.fractalType < 2;
    gl.uniform1i(programInfo.uniformLocations.highPrecision, highPrecision ? 1 : 0);

    gl.activeTexture(gl.TEXTURE0);
    gl.bindTexture(gl.TEXTURE_2D, paletteTexture);
    gl.uniform1i(programInfo.uniformLocations.paletteTexture, 0);

    gl.drawArrays(gl.TRIANGLE_STRIP, 0, 4);

    updateStats();

    // Track RAF ID for pause/resume during video export
    if (!isExporting) {
        mainRenderRAF = requestAnimationFrame(drawScene);
    }
}
```

Note: the old `drawScene` contained a `logPerformanceData()` call inside the FPS block — it is intentionally gone (the whole perf-logging system is deleted in B14).

**Step 3.** Replace the **entire** `resizeCanvasToDisplaySize` function with:

```js
function resizeCanvasToDisplaySize(canvas, fullRes) {
    // While interacting, cap DPR for frame rate; the settled frame renders at
    // native DPR (capped at 3) so the final image is sharp on mobile screens.
    const screenCtx = getScreenContext();
    const native = window.devicePixelRatio || 1;
    const interactionCap = screenCtx.isMobile ? 1.5 : 2.0;
    const dpr = fullRes ? Math.min(native, 3.0) : Math.min(native, interactionCap);

    const displayWidth = Math.round(canvas.clientWidth * dpr);
    const displayHeight = Math.round(canvas.clientHeight * dpr);

    if (canvas.width !== displayWidth || canvas.height !== displayHeight) {
        canvas.width = displayWidth;
        canvas.height = displayHeight;
        return true;
    }
    return false;
}
```

**Step 4 — pan-scale fix (required by the variable DPR).** Pan code divides CSS-pixel finger deltas by the *device-pixel* canvas height, so pan speed currently varies with DPR (and would change between interaction and idle after Step 3). Run `grep -n "state.zoomSize / canvas.height" script.js` — there are exactly **two** occurrences (one in the mousemove RAF handler, one in the touchmove RAF handler). Change both to:

```js
                const scale = state.zoomSize / canvas.clientHeight;
```

**Step 5 — wake the loop on every UI-driven state change.** Run:

```bash
grep -n "state.paletteId =\|state.maxIterations =\|state.fractalType =\|state.zoomCenter =\|state.targetZoomCenter =\|state.targetZoomSize =" script.js
```

For every assignment that is inside a **UI event handler or init/restore function** (palette-card clicks, iterations slider, fractal-card clicks, `resetView`, `restoreStateFromExport`, the tab/catalogue handlers — NOT the ones inside `drawScene`, `handleZoom`, the touch/mouse handlers, or `startHypnoticJourney`'s animate loop, which all run while the loop is already awake — though adding `requestRender()` there too is harmless), add a `requestRender();` call at the end of that handler. When in doubt, add it — a spurious `requestRender()` costs one frame; a missing one freezes the canvas.

**Step 6.** In `resumeMainRender` (anchor: `function resumeMainRender`), add `needsRender = true;` as the first line of the function body.

**Verify:** `node --check script.js`. In the browser: interact, then let go — within ~1s the image refines (sharper on a retina screen) and `chrome://gpu`-level GPU usage drops to ~0 (check via browser task manager). Open the console and confirm `document.getElementById('glCanvas').width === Math.round(innerWidth * Math.min(devicePixelRatio, 3))` after idling. Changing palette/iterations/fractal updates the image immediately.

**Commit:** `Render on demand: idle GPU sleep + full-DPR refine on settle, DPR-correct panning`

## B3. Pinch-pan + two-to-one finger handoff (audit #11)

**Step 1.** Anchor: `let lastTouchDistance = 0;`. Replace with:

```js
let lastTouchDistance = 0;
let lastPinchCenter = null;
```

**Step 2.** In the `touchstart` listener, the two-finger branch (anchor: `lastTouchDistance = Math.sqrt(dx * dx + dy * dy);`) — directly after that line, add:

```js
        lastPinchCenter = {
            x: (e.touches[0].clientX + e.touches[1].clientX) / 2,
            y: (e.touches[0].clientY + e.touches[1].clientY) / 2
        };
```

And in the same listener's one-finger branch (anchor: `state.velocity = { x: 0, y: 0 };` inside `touchstart`), add after it:

```js
        lastPinchCenter = null;
```

**Step 3.** In the `touchmove` RAF handler's two-finger branch — anchor:

```js
                const centerX = (touches[0].clientX + touches[1].clientX) / 2;
                const centerY = (touches[0].clientY + touches[1].clientY) / 2;
```

directly **after** those two lines and **before** `if (lastTouchDistance > 0) {`, insert (this is the pinch-pan — both fingers moving together now translates the view, like every map app):

```js
                if (lastPinchCenter) {
                    const pdx = centerX - lastPinchCenter.x;
                    const pdy = centerY - lastPinchCenter.y;
                    const panScale = state.targetZoomSize / canvas.clientHeight;
                    state.targetZoomCenter.x -= pdx * panScale;
                    state.targetZoomCenter.y += pdy * panScale;
                }
                lastPinchCenter = { x: centerX, y: centerY };
```

**Step 4.** Replace the **entire** `touchend` listener (anchor: `canvas.addEventListener('touchend', (e) => {`) with:

```js
canvas.addEventListener('touchend', (e) => {
    if (e.touches.length === 1) {
        // Pinch ended with one finger still down: hand off to single-finger pan
        state.isDragging = true;
        state.lastMouse = { x: e.touches[0].clientX, y: e.touches[0].clientY };
        state.velocity = { x: 0, y: 0 };
        lastTouchDistance = 0;
        lastPinchCenter = null;
    } else if (e.touches.length === 0) {
        state.isDragging = false;
        lastTouchDistance = 0;
        lastPinchCenter = null;
    }
});
```

The `state.lastMouse` re-seed is mandatory — without it the first pan frame after the handoff jumps.

**Verify:** on a touch device or DevTools touch emulation: two-finger drag pans; pinch zooms anchored at the midpoint; lift one finger and keep moving — panning continues seamlessly.

**Commit:** `Touch: pinch-pan via midpoint translation, seamless 2-to-1 finger handoff`

## B4. Analog wheel zoom (audit flagged item, confirmed)

`handleZoom` already supports analog magnitude (`const speedMultiplier = Math.abs(delta);`) but the wheel listener throws it away with `Math.sign`. Replace the wheel listener (anchor: `canvas.addEventListener('wheel', (e) => {`) with:

```js
canvas.addEventListener('wheel', (e) => {
    e.preventDefault();
    // Preserve analog magnitude: trackpads send many small deltas, mice ~100/tick.
    // deltaMode 1 = lines (Firefox mice), convert to pixel-ish units.
    const unit = e.deltaMode === 1 ? 16 : 1;
    const delta = Math.max(-2.5, Math.min(2.5, (e.deltaY * unit) / 100));
    if (delta !== 0) handleZoom(delta, e.clientX, e.clientY);
}, { passive: false });
```

(A standard mouse wheel tick has `deltaY ≈ 100` → `delta = 1.0`, identical to today's behavior; trackpad gestures become proportional instead of full-speed.)

**Verify:** slow two-finger trackpad scroll zooms slowly; a fast flick zooms fast; mouse wheel feels unchanged.

**Commit:** `Wheel zoom: use analog delta magnitude instead of sign`

## B5. Depth limit with feedback (audit #12 minimal fix — NO perturbation)

**Step 1.** Directly above `function handleZoom(delta, x, y) {`, insert:

```js
function clampZoomDepth() {
    // Double-single emulation breaks down near zoomSize 1e-13; Sierpinski has
    // no high-precision path and pixelates near 1e-5. Stop before the mush.
    const minZoomSize = state.fractalType < 2 ? 1e-13 : 1e-5;
    if (state.targetZoomSize < minZoomSize) {
        state.targetZoomSize = minZoomSize;
        if (!state.depthLimitNotified) {
            state.depthLimitNotified = true;
            showToast('Maximum depth reached');
        }
    } else if (state.targetZoomSize > minZoomSize * 10) {
        state.depthLimitNotified = false;
    }
}
```

**Step 2.** In `handleZoom`, directly after the zoom apply block (anchor: the `if (delta > 0) {` / `state.targetZoomSize *= zoomFactor;` / `} else {` / `state.targetZoomSize /= zoomFactor;` / `}` block), add a line: `clampZoomDepth();`

**Step 3.** In the `touchmove` pinch branch, directly after its own zoom apply block (anchor: `state.targetZoomSize /= (1 - delta * zoomStrength);` — add after the closing `}` of that if/else), add: `clampZoomDepth();`

**Step 4.** Add `depthLimitNotified: false,` to the `state` object (anchor: `juliaC: { x: -0.7269, y: 0.1889 },` — insert the new property on the next line).

**Verify:** hold zoom-in on the circle control — zoom stops at the wall, one toast appears, image stays crisp; zoom out a bit and back in — no repeated toast spam.

**Commit:** `Clamp zoom at the precision wall with a max-depth toast`

## B6. fractalType travels with scenes end-to-end (audit #2)

**Step 1.** In `saveScene`, in the `newLoc` object (anchor: `// 2. Save to LocalStorage (Catalogue)`), add after the `paletteId: state.paletteId,` line:

```js
        fractalType: state.fractalType,
```

**Step 2.** In `showAddLocationModal`'s confirm handler, in its `newLoc` object (anchor: `paletteId: locationData.paletteId,` inside `confirmBtn.onclick`), add after that line:

```js
            fractalType: locationData.fractalType,
```

**Step 3.** In `startHypnoticJourney(loc)`, directly after the line `state.velocity = { x: 0, y: 0 };`, insert:

```js
    // Switch fractal if this location belongs to a different one
    if (loc.fractalType !== undefined && loc.fractalType !== state.fractalType) {
        state.fractalType = loc.fractalType;
        document.querySelectorAll('.fractal-card').forEach(c => c.classList.remove('active'));
        const fractalCard = document.querySelector(`.fractal-card[data-type="${loc.fractalType}"]`);
        if (fractalCard) fractalCard.classList.add('active');
        renderCatalogue();
    }
```

**Step 4.** In `renderCatalogue`, replace (anchor):

```js
    // Combine predefined and saved locations
    const allLocations = [...currentLocations, ...savedLocations];
```

with:

```js
    // Combine predefined and saved locations (legacy saves without a
    // fractalType are shown everywhere)
    const relevantSaved = savedLocations.filter(
        (l) => l.fractalType === undefined || l.fractalType === state.fractalType
    );
    const allLocations = [...currentLocations, ...relevantSaved];
```

(`shareLocation` already picks up `loc.fractalType ?? state.fractalType` via B1's helper.)

**Verify:** switch to Julia, save a scene, switch to Mandelbrot — the Julia scene is not listed under Mandelbrot; switch back to Julia, click it — journey renders Julia. Share it, open the URL in a private window, accept — lands in Julia.

**Commit:** `Store fractalType with scenes; journeys and shares restore it`

## B7. Escape user text in the catalogue (audit #3 — security)

In `renderCatalogue`'s template (anchor: `<h4>${loc.title}</h4>`), change:

```js
                <h4>${loc.title}</h4>
                <p>${loc.description}</p>
```

to:

```js
                <h4>${escapeHtml(loc.title)}</h4>
                <p>${escapeHtml(loc.description)}</p>
```

Then run `grep -n "innerHTML" script.js` — confirm the only remaining `innerHTML` uses are `container.innerHTML = '';` (safe) and this template (now escaped). If any other usage interpolates user-controlled data, escape it the same way and report it.

**Verify:** open `http://localhost:8000/?x=0&y=0&z=2&n=%3Cimg%20src%3Dx%20onerror%3Dalert(1)%3E`, accept the trip, open the catalogue — the name renders as literal text, no alert fires.

**Commit:** `Escape user-supplied titles/descriptions in catalogue HTML (XSS fix)`

## B8. Validate inbound URL params (audit #5)

In `showAddLocationModal`, replace the `locationData` object literal (anchor: `// Extract location data from URL` through the closing `};`) with:

```js
    // Extract and validate location data from URL
    const locationData = {
        x: parseFloat(params.get('x')),
        y: parseFloat(params.get('y')),
        z: parseFloat(params.get('z')),
        zoom: parseFloat(params.get('z')),
        iterations: Math.min(Math.max(parseInt(params.get('i'), 10) || 500, 100), 2000),
        paletteId: Math.min(Math.max(parseInt(params.get('p'), 10) || 0, 0), 9),
        fractalType: Math.min(Math.max(parseInt(params.get('f'), 10) || 0, 0), 2),
        name: (params.get('n') || 'this location').slice(0, 80),
        duration: Math.min(Math.max(parseInt(params.get('d'), 10) || 30, 3), 300)
    };

    if (![locationData.x, locationData.y, locationData.z].every(Number.isFinite)
        || locationData.z <= 0) {
        window.history.replaceState({}, document.title, window.location.pathname);
        return;
    }
```

**Verify:** `?x=abc&y=0&z=5` → no modal, app loads home view, no console errors. A valid link still shows the modal.

**Commit:** `Validate and clamp inbound shared-link parameters`

## B9. Declining a shared trip jumps to the location instead of discarding it (audit #6b)

In `showAddLocationModal`, replace the entire `cancelBtn.onclick = () => { ... };` block (anchor: `// Handle cancel`) with:

```js
    // Handle cancel: jump straight to the shared view, without saving or animating
    cancelBtn.onclick = () => {
        modal.classList.add('hidden');

        state.fractalType = locationData.fractalType;
        document.querySelectorAll('.fractal-card').forEach(c => c.classList.remove('active'));
        const fractalCard = document.querySelector(`.fractal-card[data-type="${locationData.fractalType}"]`);
        if (fractalCard) fractalCard.classList.add('active');

        state.paletteId = locationData.paletteId;
        document.querySelectorAll('.palette-card').forEach(c => c.classList.remove('active'));
        const paletteBtn = document.querySelector(`.palette-card[data-palette="${locationData.paletteId}"]`);
        if (paletteBtn) paletteBtn.classList.add('active');

        state.maxIterations = locationData.iterations;
        const iterationsEl = document.getElementById('iterations');
        const iterValueEl = document.getElementById('iterValue');
        if (iterationsEl) iterationsEl.value = locationData.iterations;
        if (iterValueEl) iterValueEl.innerText = locationData.iterations;

        state.zoomCenter = { x: locationData.x, y: locationData.y };
        state.targetZoomCenter = { x: locationData.x, y: locationData.y };
        state.zoomSize = 3.0 / locationData.zoom;
        state.targetZoomSize = state.zoomSize;
        renderCatalogue();
        requestRender();

        // Clear URL params to prevent showing again on refresh
        window.history.replaceState({}, document.title, window.location.pathname);
    };
```

**Verify:** open a shared link, click "Jump there instantly" — you land on the exact view immediately; nothing is added to the catalogue; refreshing shows the home view.

**Commit:** `Shared-trip decline now jumps to the location instead of discarding it`

## B10. No spurious toast when the user cancels the share sheet (audit #6a)

In `shareLocation`, change the `navigator.share({...}).catch(() => {` handler (anchor: `}).catch(() => {` directly after the `navigator.share({` block) to:

```js
        }).catch((err) => {
            if (err && err.name === 'AbortError') return; // user closed the share sheet
            // Fallback to clipboard
            navigator.clipboard.writeText(shareText).then(() => {
                showToast('Link copied to clipboard!');
            }).catch(() => {
                showToast('Could not share');
            });
        });
```

**Verify:** on a device with the native share sheet, open it and dismiss it — no toast appears.

**Commit:** `Ignore AbortError when user cancels the native share sheet`

## B11. Import mp4-muxer locally (audit #7, pairs with A1)

Anchor: `await import('https://unpkg.com/mp4-muxer@5.2.2/build/mp4-muxer.mjs')`. Replace that line with:

```js
            const module = await import('./mp4-muxer.mjs');
```

**Verify:** after A1 exists, export a short video — it completes with no network request to unpkg (check DevTools Network); then repeat with DevTools offline (after one online page load) — still works.

**Commit:** `Import vendored mp4-muxer instead of unpkg CDN`

## B12. Survive WebGL context loss (audit flagged item)

**Step 1.** Find the global `state` object (anchor: `const state = {` containing `juliaC: { x: -0.7269`). Directly **after** the statement's closing `};`, insert:

```js
// Restore the view after a WebGL context-loss reload
try {
    const ctxSnapshot = sessionStorage.getItem('fractonaut_ctx_restore');
    if (ctxSnapshot) {
        sessionStorage.removeItem('fractonaut_ctx_restore');
        const s = JSON.parse(ctxSnapshot);
        if ([s.x, s.y, s.z].every(Number.isFinite) && s.z > 0) {
            state.zoomCenter = { x: s.x, y: s.y };
            state.targetZoomCenter = { x: s.x, y: s.y };
            state.zoomSize = 3.0 / s.z;
            state.targetZoomSize = state.zoomSize;
            if (Number.isFinite(s.i)) state.maxIterations = s.i;
            if (Number.isFinite(s.p)) state.paletteId = s.p;
            if (Number.isFinite(s.f)) state.fractalType = s.f;
        }
    }
} catch (err) {
    // Corrupted snapshot — ignore and start fresh
}
```

(If `targetZoomCenter`/`targetZoomSize` are initialized as part of the `state` literal, this assignment after the fact is still correct.)

**Step 2.** Next to the `wheel` listener (anchor: `canvas.addEventListener('wheel'`), add **before** it:

```js
// A lost GL context leaves the canvas permanently black; snapshot the view and
// reload when the browser hands the context back.
canvas.addEventListener('webglcontextlost', (e) => {
    e.preventDefault();
    sessionStorage.setItem('fractonaut_ctx_restore', JSON.stringify({
        x: state.zoomCenter.x, y: state.zoomCenter.y, z: 3.0 / state.zoomSize,
        i: state.maxIterations, p: state.paletteId, f: state.fractalType
    }));
});
canvas.addEventListener('webglcontextrestored', () => location.reload());
```

Known cosmetic limitation (acceptable): after a restore-reload, the palette/iteration *UI controls* may show defaults while the view is correct.

**Verify:** in the console run `document.getElementById('glCanvas').getContext('webgl2').getExtension('WEBGL_lose_context').loseContext()` — page snapshots and (on `restoreContext()` or browser-initiated restore) reloads into the same view.

**Commit:** `Handle WebGL context loss: snapshot view and restore after reload`

## B13. Gate export resolutions by GPU limits (audit #15)

**Step 1.** In `initExportResolutionModal` (anchor: `function initExportResolutionModal`), add at the top of the function body:

```js
    // Disable resolutions the GPU cannot render (16K exceeds most limits)
    const maxGpuSize = Math.min(
        gl.getParameter(gl.MAX_TEXTURE_SIZE),
        gl.getParameter(gl.MAX_RENDERBUFFER_SIZE)
    );
    document.querySelectorAll('#exportResolutionModal .resolution-option').forEach((btn) => {
        if (parseInt(btn.dataset.width, 10) > maxGpuSize) {
            btn.disabled = true;
            btn.classList.add('unsupported');
            btn.title = `Your GPU supports up to ${maxGpuSize}px`;
        }
    });
```

**Step 2.** In `renderHighResolutionExport`, after the export GL context is created (anchor: search for where `exportGl` is first assigned via `getContext`), add a runtime guard:

```js
            const exportMax = Math.min(
                exportGl.getParameter(exportGl.MAX_TEXTURE_SIZE),
                exportGl.getParameter(exportGl.MAX_RENDERBUFFER_SIZE)
            );
            if (exportWidth > exportMax || exportHeight > exportMax) {
                reject(new Error(`This device supports exports up to ${exportMax}px`));
                return;
            }
```

(Adapt `reject(...)` to however that function signals errors — it is a `new Promise((resolve, reject) =>` body; if a cleanup path exists, route through it.)

**Verify:** in the console, `gl.getParameter(gl.MAX_RENDERBUFFER_SIZE)` — if < 15360, the 16K button is greyed out with a tooltip.

**Commit:** `Disable export resolutions beyond GPU texture/renderbuffer limits`

## B14. Delete the dead performance-logging system (audit #13)

1. Delete these entire functions (anchors: `function startPerformanceLogging()`, `function logPerformanceData()`, `function stopPerformanceLogging()`), the commented-out `// function downloadPerformanceCSV() {` block that follows them, and the commented call `//startPerformanceLogging();`.
2. Run `grep -n "perfLogs\|isTestMode\|perfTimer\|PerformanceLogging\|logPerformanceData" script.js` and remove every remaining reference (state properties like `perfLogs: [],`, any `isTestMode`/`perfTimer` fields and reads). B2's new `drawScene` already dropped the only runtime call.
3. Re-run the grep — zero hits.

**Verify:** `node --check script.js`; app loads and renders with no console errors.

**Commit:** `Remove dead performance-logging system`

---

# Acceptance checklist (Verifier)

Run after both streams are merged. Serve with `python3 -m http.server 8000`.

| # | Check | Pass condition |
|---|-------|----------------|
| 1 | `node --check script.js && node --check sw.js` | exits 0 |
| 2 | `grep -c "toFixed(6)" script.js` | only hits are inside `updateStats` |
| 3 | `grep -rn "unpkg\|googleapis\|gstatic" index.html script.js sw.js style.css` | zero hits |
| 4 | Deep-zoom share round-trip: zoom until stats show ≥ 10⁷×, save a scene, share it, open the URL in a private window, accept | recipient view identical (same structures on screen) |
| 5 | Julia round-trip: save a Julia scene, switch to Mandelbrot, reopen the saved scene's share URL | renders Julia, not Mandelbrot |
| 6 | XSS: open `/?x=0&y=0&z=2&n=%3Cimg%20src%3Dx%20onerror%3Dalert(1)%3E`, accept, open catalogue | no alert; name shows as literal text |
| 7 | Malformed link `/?x=abc&y=0&z=5` | no modal, home view, no console errors |
| 8 | Idle GPU: load, wait 5 s without input, check browser task manager GPU usage | near 0%; rises again on interaction |
| 9 | Idle sharpness: after 2 s idle run `document.getElementById('glCanvas').width` | equals `Math.round(canvas.clientWidth × min(devicePixelRatio, 3))` |
| 10 | Touch (device or emulation): two-finger drag pans; pinch zooms at midpoint; lift one finger mid-pinch and keep moving | pan continues without jump |
| 11 | Trackpad slow scroll vs fast flick | proportional zoom speed |
| 12 | Hold zoom-in to the wall | zoom stops, single "Maximum depth reached" toast, image stays crisp |
| 13 | Offline: load once online, DevTools → offline, reload; then export a video offline | app loads with fonts; video export succeeds |
| 14 | Export modal | resolutions above `gl.getParameter(gl.MAX_RENDERBUFFER_SIZE)` are greyed out |
| 15 | Decline a shared trip ("Jump there instantly") | lands on exact view, nothing saved, refresh shows home |
| 16 | Cancel the native share sheet (mobile) | no "copied" toast |
| 17 | Console clean | no errors during any of the above |

Finally: update the `Recent Updates` section of `claude.md` with one line per shipped fix, and append `— FIXED <date>` to the matching items in `AUDIT.md`.

---

# Explicitly OUT of scope

- **Perturbation-theory rendering** — excluded by the project owner. The depth clamp (B5) is the only depth-related change. Do not add it, do not add libraries for it.
- **Export-path consolidation** (`createExportContext()` refactor from audit #13): regression risk outweighs the benefit in this pass. Skip.
- **`juliaC` URL parameter**: the Julia constant is currently a fixed compile-time value that never changes at runtime, so it does not need to be shared. If a Julia-seed picker is ever added, add a `j` param to `buildLocationParams` and the URL parser.
- **Optional, only if everything above is verified and time remains:**
  - Adaptive stats precision: in `updateStats`, scale displayed decimals with zoom: `const digits = Math.max(6, Math.min(15, Math.ceil(-Math.log10(state.zoomSize)) + 4));` and use `.toFixed(digits)`.
  - Verify the precision-mode switch visually: zoom slowly through ~3000× (zoomSize ≈ 0.001) and watch for a one-frame shift; if visible, change `0.001` to `0.0003` in **both** `highPrecision` expressions (main draw + export path).
  - Unify fling-velocity multipliers (mouse `0.5` vs touch `2.0`) if mouse flings feel dead.
