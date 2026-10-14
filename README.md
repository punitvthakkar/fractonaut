# Fractonaut

An interactive fractal explorer. Live at [fractonaut.com](https://fractonaut.com).

Explore the Mandelbrot set, Julia set and Sierpinski triangle in real time with WebGL2, zoom into deep detail, pick from 10 color palettes, save and share locations, and export screenshots or fly-through videos.

## Files

- `index.html` — the whole app (markup, styles and script)
- `manifest.json`, `256.png`, `512.png`, `1024.png` — PWA manifest and icons
- `sw.js` — service worker for offline use
- `CNAME` — custom domain for GitHub Pages

## Run locally

```sh
python3 -m http.server 8080
```

Then open http://localhost:8080. Requires a browser with WebGL2.
