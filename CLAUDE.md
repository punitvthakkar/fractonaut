# CLAUDE.md - Fractonaut Project Guide

## Project Overview

**Fractonaut** is an interactive web-based fractal explorer that allows users to dive into the mesmerizing world of fractals. It's a Progressive Web App (PWA) that runs entirely in the browser using WebGL2 for high-performance rendering.

- **Live URL**: https://fractonaut.com
- **Type**: Progressive Web App (PWA)
- **Primary Language**: Vanilla JavaScript (ES6+)
- **Rendering**: WebGL2 with GLSL ES 3.00 shaders
- **Total Lines of Code**: ~2,931 JS + ~1,400+ CSS

## Architecture

### File Structure

```
/
├── index.html         (21KB)  - Main HTML structure and UI elements
├── script.js          (106KB) - Core application logic and WebGL rendering
├── style.css          (53KB)  - Material Design 3 styling and animations
├── sw.js              (2.2KB) - Service Worker for offline capability
├── manifest.json      (282B)  - PWA manifest
├── 256.png                    - App icon (256x256)
├── 512.png                    - App icon (512x512)
├── 1024.png                   - App icon (1024x1024)
└── CNAME                      - GitHub Pages custom domain

```

### Technology Stack

- **Frontend**: Vanilla JavaScript (no frameworks)
- **Graphics**: WebGL2 API
- **Shaders**: GLSL ES 3.00 (OpenGL ES 3.0 Shading Language)
- **Styling**: Custom CSS with Material Design 3 design tokens
- **Font**: Google Fonts - Outfit (300, 400, 600, 700 weights)
- **PWA**: Service Worker with network-first caching strategy
- **Build**: No build process - direct file serving

## Core Features

### 1. Fractal Types (script.js:343)

Three distinct fractal types are supported:

- **Mandelbrot Set** (`fractalType: 0`) - The classic fractal
- **Julia Set** (`fractalType: 1`) - Self-similar spiral patterns
- **Sierpinski Triangle** (`fractalType: 2`) - Infinite fractal triangles

### 2. Rendering Engine

#### WebGL2 Context (script.js:295-302)
- Uses optimized context settings for performance
- Disables unnecessary features (alpha, depth, stencil buffers)
- Double-precision emulation in shaders for deep zoom levels

#### Shader Architecture (script.js:2-292)

**Vertex Shader**: Simple pass-through for full-screen quad
**Fragment Shader**: Contains:
- Double-precision arithmetic emulation (ds_add, ds_sub, ds_mul, ds_sqr)
- Three fractal algorithms
- 10 color palettes (0-9)
- Smooth iteration coloring using logarithmic smoothing
- Cardioid optimization for Mandelbrot set

### 3. State Management (script.js:325-358)

Global `state` object tracks:
- Camera position (`zoomCenter`, `zoomSize`)
- Rendering parameters (`maxIterations`, `paletteId`)
- Interaction state (`isDragging`, `velocity`)
- Performance metrics (`currentFps`, `perfLogs`)
- UI state (`panelVisible`, `tutorialShown`)
- Export settings (`exportResolution`)

### 4. Location Catalogue (script.js:360-487)

Predefined scenic locations for each fractal type:
- 4 locations per fractal type
- Each location stores: coordinates (x, y), zoom level, iterations, palette
- Organized as `locations[fractalType][index]`

### 5. Interactive Controls

#### Touch & Mouse Input
- **Pan**: Drag to move around
- **Zoom**: Pinch-to-zoom (mobile) or scroll wheel (desktop)
- **Circle Control**: Central circular slider for zoom/detail adjustment (index.html:68-78)

#### UI Elements (index.html)
- **Control Panel** (line 81-239): Slide-in panel with settings
- **Reset Button** (line 35-42): Return to home view
- **Save Button** (line 45-50): Save current location
- **Video Export Button** (line 53-58): Export video flythroughs
- **Location Info Overlay** (line 244-247): Displays current location name

### 6. Export Capabilities

#### Image Export
- Supports 4K, 8K, and 16K resolution exports
- Uses offscreen canvas for high-resolution rendering
- Progress tracking with visual feedback

#### Video Export (script.js:2400+)
- HD (720p) and FHD (1080p) support
- 30 FPS or 60 FPS options
- Frame-by-frame rendering with progress indicators
- Video encoding using MediaRecorder API

### 7. PWA Features (sw.js)

**Service Worker Strategy**:
- Install event: Precache core assets
- Fetch event: Network-first with cache fallback
- Update on every launch when online
- Offline capability for cached resources

**Caching**:
- `fractonaut-v1`: Static assets cache
- `fractonaut-runtime`: Runtime cache for dynamic resources

## Design System

### Material Design 3 Tokens (style.css:1-93)

The app uses a comprehensive M3 design system:

**Color Palette**:
- Surface colors: `--md-sys-color-surface`, `--md-sys-color-surface-container`
- Primary: `--md-sys-color-primary` (#D0BCFF - purple)
- Text: `--md-sys-color-on-surface`, `--md-sys-color-on-surface-variant`

**Motion**:
- Easing: `cubic-bezier(0.2, 0.0, 0, 1.0)` - emphasized motion
- Durations: 200ms (fast), 400ms (normal), 600ms (slow)

**Shapes**:
- Border radius system from 4px (extra-small) to 28px (extra-large)
- Full rounded corners: 9999px

**Elevations**:
- Three elevation levels with layered shadows
- Glow effects for interactive elements

## Development Workflows

### Making Changes

#### 1. Modifying Shaders (script.js:2-292)

When editing GLSL shaders:
- Maintain GLSL ES 3.00 syntax (`#version 300 es`)
- Use `highp` precision for floats
- Test double-precision functions for deep zoom accuracy
- Remember: WebGL2 has loop iteration limits - use dynamic breaks

**Common Shader Modifications**:
```javascript
// Adding a new color palette (script.js:254-286)
// 1. Add new palette ID in the if-else chain
// 2. Define palette parameters using palette() function
// 3. Update palette selection UI in index.html
```

#### 2. Adding New Fractals

To add a new fractal type:
1. Increment `u_fractalType` in shader uniforms
2. Add fractal algorithm in fragment shader main()
3. Add location presets in `locations` object (script.js:360-487)
4. Create UI card in index.html fractal grid (line 180-203)
5. Update fractal switching logic in event handlers

#### 3. UI Modifications (index.html + style.css)

**Component Structure**:
- UI uses BEM-like naming: `.component-element--modifier`
- All interactive elements have ARIA labels
- Mobile-first responsive design with breakpoints at 768px

**CSS Organization**:
1. CSS variables (root)
2. Reset & base styles
3. Layout components
4. Interactive controls
5. Modals & overlays
6. Media queries

#### 4. Performance Optimization

**Current Optimizations**:
- RAF (RequestAnimationFrame) throttling for interactions
- Canvas doesn't preserve drawing buffer
- Viewport-based iteration count (lower on mobile)
- Cardioid check for Mandelbrot set (script.js:220-228)

**Performance Monitoring** (script.js:350-357):
```javascript
state.isTestMode: true    // Enable logging
state.perfLogs: []        // Performance data array
state.currentFps: 0       // Real-time FPS
```

### Testing Checklist

When making changes, test:

1. **Desktop Browsers**:
   - Chrome/Edge (WebGL2 support)
   - Firefox
   - Safari (WebKit-specific prefixes)

2. **Mobile Devices**:
   - Touch controls (pan, pinch-to-zoom)
   - Performance with lower iteration counts
   - PWA installation flow

3. **Rendering**:
   - All three fractal types
   - All 10 color palettes
   - Deep zoom levels (test double-precision)
   - Smooth animation at 60fps

4. **Export**:
   - Screenshot capture
   - Video rendering (check memory usage)
   - High-resolution exports (4K, 8K, 16K)

5. **PWA**:
   - Offline functionality
   - Install prompt
   - Service worker updates

### Common Issues & Solutions

#### Issue: Low FPS on mobile
**Solution**: Reduce `state.maxIterations` (script.js:328). Mobile defaults to 300 vs desktop 500.

#### Issue: Precision artifacts at deep zoom
**Solution**: Enable `u_highPrecision` mode which uses double-precision emulation in shaders.

#### Issue: Export crashes on high resolution
**Solution**: Limit resolution based on device capabilities. Check `gl.MAX_TEXTURE_SIZE`.

#### Issue: Service worker not updating
**Solution**:
1. Check cache names in sw.js (CACHE_NAME, RUNTIME_CACHE)
2. Verify `skipWaiting()` is called on install
3. Clear browser cache and hard reload

## Git Workflow

### Branch Strategy
- Main branch: Production-ready code
- Feature branches: `claude/feature-name-sessionid` format
- Always develop on designated feature branch

### Commit Guidelines
- Use descriptive commit messages
- Prefix: `feat:`, `fix:`, `style:`, `refactor:`, `perf:`, `docs:`
- Example: `feat: add new fractal type - burning ship`

### Deployment
This project is deployed via GitHub Pages:
1. Push to main branch
2. GitHub Pages automatically serves from root
3. CNAME file ensures custom domain (fractonaut.com)

## Key Conventions

### 1. Code Style

**JavaScript**:
- Use `const` for immutable bindings, `let` for mutable
- Avoid `var`
- Camel case for variables: `zoomCenter`, `maxIterations`
- Function declarations for named functions
- Arrow functions for callbacks

**CSS**:
- Use CSS custom properties for themeable values
- Mobile-first media queries
- Prefer `rem` for spacing, `px` for borders
- Use `will-change` sparingly for GPU acceleration

### 2. Naming Patterns

**WebGL Uniforms**: `u_` prefix (e.g., `u_resolution`, `u_zoomCenter_x`)
**Shader Functions**: `ds_` prefix for double-precision (e.g., `ds_add`, `ds_mul`)
**State Properties**: Descriptive names (e.g., `isDragging`, `targetZoomCenter`)
**DOM IDs**: Camel case (e.g., `glCanvas`, `controlPanel`, `resetBtn`)

### 3. Performance Guidelines

- Minimize shader uniform updates (batch when possible)
- Use RAF for all animations
- Avoid layout thrashing (batch DOM reads/writes)
- Debounce expensive operations (resize, etc.)
- Monitor FPS and adjust iteration count dynamically

### 4. Accessibility

- All interactive elements must have ARIA labels
- Support keyboard navigation where applicable
- Maintain sufficient color contrast (WCAG AA)
- Provide text alternatives for visual information

### 5. Browser Compatibility

**Required**:
- WebGL2 support (check `canvas.getContext('webgl2')`)
- ES6+ features (const/let, arrow functions, template literals)
- CSS custom properties
- Service Worker API

**Fallbacks**:
- Detect WebGL2 unavailability and show error
- Provide button-based controls as alternative to gestures
- Desktop/mobile responsive layouts

## URLs & Resources

- **Live Site**: https://fractonaut.com
- **Repository**: Check git remote origin
- **Font**: https://fonts.google.com/specimen/Outfit
- **WebGL2 Reference**: https://developer.mozilla.org/en-US/docs/Web/API/WebGL2RenderingContext
- **GLSL ES 3.00**: https://www.khronos.org/files/opengles_shading_language.pdf

## Quick Reference

### State Reset
```javascript
state.zoomCenter = { x: -0.74364388703, y: 0.1318259042 };
state.zoomSize = 3.0;
state.fractalType = 0; // Mandelbrot
```

### Trigger Render
```javascript
requestAnimationFrame(drawScene);
```

### Change Fractal
```javascript
state.fractalType = 1; // 0=Mandelbrot, 1=Julia, 2=Sierpinski
```

### Access Canvas/GL
```javascript
const canvas = document.getElementById('glCanvas');
const gl = canvas.getContext('webgl2');
```

## Notes for AI Assistants

1. **No Build Process**: This is a vanilla JS project - edit files directly, no transpilation needed.

2. **Shader Edits Require Reload**: Changes to shader source strings require page refresh to recompile.

3. **State is Global**: The `state` object is the single source of truth - modify it carefully.

4. **Performance Critical**: This app renders 60fps animations - test performance impact of any changes.

5. **Mobile Matters**: Always test mobile responsiveness and touch interactions.

6. **PWA Caching**: Remember to update service worker cache version when changing assets.

7. **Math Precision**: Deep zoom requires double-precision - test at zoom levels > 10,000.

8. **Export is Memory-Intensive**: High-resolution exports can crash on low-memory devices - add safeguards.

---

**Last Updated**: 2025-12-07
**Version**: 1.0
