# Fractonaut Audit — June 12, 2026

Audit of the codebase (commit `a17cadc`) against the four guiding principles:
1. Minimal external libraries
2. Minimal code overhead
3. Extremely smooth, beautiful, and precise rendering
4. Sharing options to send friends to exact locations

All issues marked **confirmed** were verified against the actual source.

---

## Principle 4 — Sharing

### 1. Share URLs destroy deep-zoom locations (confirmed, critical)
`script.js:1065`, `1216`, `1763` encode coordinates with `toFixed(6)`. Viewport width is `3/zoom`, so the ~5e-7 truncation error is visibly off-center past ~10⁴× zoom and a completely different view past ~10⁶× — while the renderer supports ~10¹²×. Locally saved scenes keep full precision (`script.js:1779` stores the raw double), but sharing a saved scene re-truncates it.
**Fix:** `x: String(loc.x)` — `String(number)` is the shortest exact round-trip representation. Best done as one shared `buildLocationParams(loc)` helper (also removes the 3× duplication).

### 2. Fractal type not saved with scenes (confirmed, major)
Saved-scene objects at `script.js:1775-1785` and `1638-1648` have no `fractalType` field; `shareLocation` encodes the *currently viewed* type (`script.js:1070`), not the location's.
**Fix:** add `fractalType` to both saved objects; share/fly with `loc.fractalType ?? state.fractalType`.

### 3. HTML injection via shared links (confirmed, security)
URL `n` param → `loc.title` → unescaped `item.innerHTML` in `renderCatalogue` (`script.js:998`), persisted in the recipient's localStorage.
**Fix:** build nodes with `textContent` or escape titles (5-line `escapeHtml()`).

### 4. No link previews (confirmed)
No `og:`/`twitter:` meta tags in `index.html` — shared links render as bare URLs in WhatsApp/iMessage/Slack.
**Fix:** static meta block (~8 lines) using the existing `1024.png` with absolute URLs.

### 5. No validation of inbound URL params (confirmed)
`script.js:1609-1612` `parseFloat`s x/y/z with no `Number.isFinite` guard → NaN center → black screen.
**Fix:** validate; skip the modal if invalid.

### 6. Sharing UX papercuts (confirmed)
- Cancelling the native share sheet triggers the clipboard fallback and a spurious "Link copied!" toast — no `AbortError` check (`script.js:1084`).
- Declining the "Start the trip?" modal discards the friend's location entirely (`script.js:1627-1631`). Better: decline = view without saving.

## Principle 1 — External libraries

### 7. mp4-muxer imported from unpkg at runtime (confirmed)
`script.js:2753`. Dynamic `import()` cannot use SRI hashes; video export depends on unpkg availability.
**Fix:** vendor `mp4-muxer.mjs` (~30 KB, MIT, single ESM file) into the repo, import locally, add to SW precache.

### 8. Google Fonts dependency (confirmed)
`index.html:21-23` — render-blocking external CSS + font fetch, not precached.
**Fix:** self-host the Outfit woff2 weights (~60 KB) with `font-display: swap`; add to precache. Also flagged (unverified): `'Space Mono'` referenced in style.css but never loaded.

## Principle 3 — Rendering

### 9. GPU never sleeps (confirmed, major)
`drawScene` unconditionally re-schedules itself (`script.js:817`) with no dirty flag — a static fractal re-renders identically at 60–120 fps forever (battery, heat, thermal throttling → jank). The shader has no time uniform, so idle frames are pixel-identical.
**Fix (~15 lines):** `needsRender` flag set by input/animation/state changes; lerp-settling logic (lines 765-771) clears it when converged. Prerequisite for #10.

### 10. Phones render at ~half native resolution (confirmed, major)
DPR capped at 1.5 on mobile (`script.js:825-826`); modern phones are DPR 2.5–3.5. Fine filaments blur.
**Fix:** after #9, render interaction at capped DPR, full-DPR refine on idle. Note: `isMobile` is a `width < 768` heuristic (narrow desktop windows get the mobile cap; portrait iPads get the desktop cap).

### 11. Multitouch feels sticky (confirmed, major)
`script.js:1933-1990`: pinch anchor math is correct, but (a) two-finger drag doesn't pan (only distance delta used); (b) lifting one finger after a pinch leaves the remaining finger dead until full lift (`isDragging` only resets at 0 touches).
**Fix:** apply midpoint translation each pinch frame; on `touchend` with 1 finger left, re-seed `state.lastMouse` and set `isDragging = true` (re-seed required or the first pan frame jumps).

### 12. No guardrail at the precision wall
Double-single arithmetic dies around `zoomSize` ~1e-13; no zoom clamp found in `handleZoom` — users hit pixelated mush with no feedback.
**Minimal fix:** clamp + "max depth" toast. Roadmap: perturbation-theory rendering (large project, unlocks ~10¹⁰⁰×).

## Principle 2 — Code overhead

### 13. Dead weight (confirmed)
- Disabled performance-logging system, ~70 lines (`script.js:~1280-1349`).
- Unused `CACHE_VERSION` (`sw.js:6`).
- URL-building block duplicated 3× (1064/1215/1762) — collapses into the helper from #1.
- Export paths duplicate shader/program setup (~843 vs ~2667) → one `createExportContext()` (medium effort).

### 14. Service worker precaches ~1.3 MB of icons (confirmed)
`sw.js:15-17` includes the 936 KB `1024.png`; manifest only references `256.png`. Also: `fetch(..., {cache: 'no-store'})` on every request = every cold start re-downloads all assets. Consider stale-while-revalidate.

## Correctness

### 15. 16K export silently fails on most hardware (confirmed)
No `gl.MAX_TEXTURE_SIZE`/`MAX_RENDERBUFFER_SIZE` check anywhere; 15360px exceeds common 8192/16384 caps.
**Fix:** query the limit, grey out unsupported options.

---

## Refuted (don't chase)
- Non-passive `touchstart` listener: harmless — `touch-action: none` (`style.css:148`) already prevents compositor blocking.

## Flagged but unverified (audit cut short by session limit)
- Wheel-delta magnitude discarded in zoom handler
- Visible pop at the 0.001 precision-mode switch
- No `webglcontextlost`/`restored` handlers (canvas goes black permanently on mobile context loss)
- Unbounded runtime cache growth in sw.js
- No progressive refinement (lower res while interacting, refine on idle)
- Inertia velocity discontinuity on mouse fling

## Suggested fix order
1. Sharing correctness: #1 + #2 + #3 (small diffs, core product promise)
2. #9 dirty-flag render loop (biggest battery/smoothness win)
3. #7 + #8 vendor dependencies
4. #10 + #11 mobile rendering and touch
5. Rest opportunistically
