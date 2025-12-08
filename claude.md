# Fractonaut

An interactive fractal explorer PWA that lets users explore and share beautiful fractal locations. Live at [fractonaut.com](https://fractonaut.com).

## Project Overview

Fractonaut is a web-based fractal visualization application that renders fractals in real-time using WebGL2. Users can explore iconic fractals like the Mandelbrot set, Julia set, and Sierpinski triangle, zoom into infinite detail, save their favorite locations, and share them with others.

## Tech Stack

- **Rendering**: WebGL2 with GLSL ES 3.0 shaders
- **Frontend**: Vanilla JavaScript (ES6+), CSS3
- **PWA**: Service Worker for offline support
- **Hosting**: GitHub Pages with custom domain

## Key Features

### Fractal Types
- **Mandelbrot Set** - The classic fractal with infinite complexity
- **Julia Set** - Self-similar spirals with configurable parameters
- **Sierpinski Triangle** - Iterated function system (IFS) fractal

### High-Precision Rendering
- Double-precision emulation in WebGL shaders for deep zooms
- Automatic precision switching based on zoom level
- Smooth iteration coloring for anti-aliased edges

### Color Palettes
10 built-in color palettes:
- Ocean, Magma, Aurora, Amber, Extreme, Neon, Golden, Cyber, Ice, Forest

### Controls
- **Circle Control**: Central joystick for zoom (up/down) and detail level (left/right)
- **Touch/Mouse**: Pinch-to-zoom, drag to pan
- **Detail Slider**: Adjust iteration count (100-2000)
- **Reset Button**: Return to home view

### Scene System
- Save locations with custom names and fly-through durations
- Animated journeys between current view and saved locations
- Share scenes via URL with encoded parameters
- Built-in scene catalogue

### Export Features
- **Screenshots**: Export current view at 4K, 8K, or 16K resolution
- **Video Export**: Render fly-through animations at HD/FHD with 30/60 FPS

### PWA Support
- Installable on mobile and desktop
- Offline support via service worker caching
- Network-first update strategy

## File Structure

```
fractonaut/
├── index.html       # Main HTML with UI components
├── script.js        # Core application logic (~2600 lines)
│   ├── WebGL shaders (GLSL)
│   ├── Fractal rendering engine
│   ├── Touch/mouse input handling
│   ├── Scene management & sharing
│   ├── Export functionality
│   └── PWA install prompt
├── style.css        # Material Design 3 inspired styling
├── sw.js            # Service worker for offline caching
├── manifest.json    # PWA manifest
├── 256.png          # App icon (256x256)
├── 512.png          # App icon (512x512)
├── 1024.png         # App icon (1024x1024)
└── CNAME            # Custom domain configuration
```

## Architecture

### Rendering Pipeline
1. Fragment shader calculates escape iterations for each pixel
2. Smooth coloring applied using logarithmic interpolation
3. Color palette lookup (procedural or texture-based)
4. High-precision mode uses split-double arithmetic when zoom > threshold

### State Management
- Global state object tracks: center position, zoom level, iterations, palette, fractal type
- URL parameters encode shareable state
- LocalStorage persists saved scenes

### Key Functions (script.js)
- `drawScene()` - Main render loop, updates uniforms and draws
- `handleZoom()` - Processes zoom input with center-point tracking
- `initCircleControl()` - Sets up joystick input handling
- `startHypnoticJourney()` - Animates fly-through to saved location
- `renderHighResolutionExport()` - Renders to offscreen canvas for export
- `shareLocation()` - Generates shareable URL with encoded state

### URL Parameters
- `x`, `y` - Center coordinates
- `z` - Zoom level
- `i` - Max iterations
- `p` - Palette ID
- `f` - Fractal type
- `n` - Location name
- `d` - Journey duration

## Development Notes

### Performance Considerations
- Canvas uses `preserveDrawingBuffer: false` for GPU optimization
- Cardioid check optimization skips iteration for points known to be in the set
- RequestAnimationFrame for smooth rendering
- `will-change` and `translateZ(0)` for GPU-accelerated UI animations

### Browser Support
- Requires WebGL2 (ES 3.0)
- Tested on modern Chrome, Firefox, Safari, Edge
- Touch events for mobile devices

### Adding New Fractals
1. Add fractal type constant and uniform in shader
2. Implement iteration logic in `main()` of fragment shader
3. Add UI card in `tabFractals` section of index.html
4. Update `fractalCards` event handling in script.js

### Adding New Palettes
1. Add palette case in shader's coloring section
2. Add palette preview CSS class (`.palette-N`)
3. Add palette card in `.palette-grid` section of index.html

## Recent Updates

- Video export with dual progress tracking (frame rendering + encoding)
- Separate video settings modal with resolution/FPS options
- Apple Liquid Glass-style UI buttons
- Improved touch responsiveness
- GPU-accelerated CSS transitions
- Service worker with network-first caching strategy
