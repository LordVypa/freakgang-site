# WITH — Codebase Reference

## Files
```
with/
  index.html   — entire site (~1990 lines, single file)
  config.json  — saved dev panel state (loaded on page open)
  scratch.png  — grunge texture (3326×2320 RGBA, present but currently unused)
```

---

## HTML Structure (index.html order)

```
<head>
  <style>          — all CSS, inline (~650 lines)
<body>
  <canvas id="dust-canvas">          z-index:5, fixed fullscreen
  <canvas id="fog-canvas">           z-index:4, fixed fullscreen, renders at 0.5× res
  <nav>
  <div class="hero">
    <div class="hero-bg">            background image placeholder
    <div class="hero-vignette">
    <div class="hero-content">
      <div class="hero-title-wrap">
        <h1 class="hero-title">      split into .jw (word) > .jl (letter) spans by JS
      </div>
      <p class="hero-desc">
      <a class="btn-wishlist">
  <section id="about">
  <div class="gallery-wrap" id="gallery">
  <section id="story">
  <section id="features">
  <section id="team">
  <section id="footer-cta">

  <script> (jitter: splits h1 into animated spans)
  <script> (scratch texture: canvas-drawn grain overlay)
  <style>  (dev panel CSS)
  <button id="dev-toggle">
  <div id="dev-panel">  (4 tabs: Title, Dust, Fog, Whisp)
  <script> (main: DEFAULTS, applyConfig, getConfig, all listeners, fog WebGL, dust, config persistence)
  <script> (whispers pool system)
```

---

## Z-index Layers
| Layer | z-index |
|---|---|
| Whisper screams (.whisper) | 6 |
| Dust canvas | 5 |
| Fog canvas | 4 |
| Hero content | 3 |
| Dev panel | 9998 |
| Dev toggle button | 9999 |

---

## CSS Sections (all in `<head><style>`)
- `/* NAV */` — fixed top nav
- `/* HERO */` — hero, hero-bg, hero-vignette, hero-content, hero-title, hero-desc, btn-wishlist
- `/* SECTION BASE */` — shared section padding/layout
- `/* ABOUT, GALLERY, STORY CHAPTERS, GAMEPLAY FEATURES, TEAM, FOOTER CTA */`
- `/* TITLE HORROR EFFECTS */` — jitter keyframes: `jt-a`, `jt-b`, `jt-c`, `jt-d`
- `/* TITLE WORD FREEZE */` — `.jw.frozen` disables animation
- `/* WHISPERS */` — `@keyframes whisp-shiver`, `.whisper` element styles
- `/* SCRATCH OVERLAY */` — `.hero-title-wrap`, `.scratch-overlay` (currently canvas grain)
- CSS vars on `:root`: `--gold: #D4B483`, `--amber`, `--black`, `--card-bg`, `--card-border`, `--text`, `--text-dim`

---

## JS Systems

### 1. Title Jitter (inline `<script>` after hero HTML)
Runs immediately. Splits `h1.hero-title` text into word spans `.jw[data-wi]` containing letter spans `.jl[data-wi][data-dur][data-del][data-anim]`. Each letter gets one of 4 CSS keyframe animations (`jt-a/b/c/d`) with random duration/delay. **After this script runs, the h1 has NO direct text nodes** — all content is in nested spans.

Per-word jitter config (hardcoded):
- "Whisper": jmin=0.96, jmax=1.65
- "in": static (jmax=0)
- "the": jmin=2.43, jmax=4.35 (slow)
- "Halls": jmin=0.61, jmax=1.22 (fastest)

### 2. Scratch Texture (second `<script>`)
Canvas-drawn grain overlay. `buildScratchTexture()` draws 60 cream diagonal lines on a 256×256 transparent canvas → data URL → set as `background-image` on `.scratch-overlay`.

`window.applyScratch(edge, tex)` — sets overlay opacity.

Dev control: single slider `#scratch-tex` (Grain opacity).

### 3. Ground Fog — WebGL (`window.FOG`)
`<canvas id="fog-canvas">` renders at 50% resolution, upscaled via CSS. GLSL ES 1.0 shader with domain-warped FBM (fractal Brownian motion), unrolled octave loops (GLSL loop bound limitation). Noise-modulated top edge eliminates hard horizontal cutoff.

**Global object:**
```js
window.FOG = {
  enabled, opacity, height, speed, color,  // '#rrggbb'
  scale, warp, flowX, flowY, octaves,       // FBM params
  throttle                                  // bool: cap at 30fps
}
```
Shader reads these every frame via `gl.uniform*`. Scroll position moves fog height via `tScroll`.

### 4. Dust Motes (`window.DUST`)
`<canvas id="dust-canvas">` canvas 2D. Particle array, each with position/velocity/size/opacity/flicker. Parallax via mouse position.

```js
window.DUST = { enabled, count, speed, opacity, size, parallax }
window.dustReinit()  // call after changing count or speed
```

### 5. Dev Panel + Config System
**IIFE, starts around line 1300.** Single source of truth for all parameters.

```js
const DEFAULTS = { /* all keys with default values */ }
function applyConfig(C)      // writes C values → DOM controls + live systems
function getConfig()         // reads DOM controls → plain object
function mergeWithDefaults(c) // Object.assign({}, DEFAULTS, c, { array copies })
```

**Persistence:**
- Chrome/Edge: File System Access API — links to `config.json` via `showOpenFilePicker`, stores handle in IndexedDB
- Firefox: `localStorage` only; "Load config.json" button (file input) shown; save downloads a file
- On panel open (Chrome): re-reads `config.json` and calls `applyConfig`

**Dev toggle bug to know:** CSS sets `#dev-panel { display:none }`. Toggle checks `p.style.display !== 'block'` (not `=== 'none'`) because inline style starts as `''`.

**Tab switching:** `.dev-tab[data-tab]` buttons toggle `.active` class on tabs and panes.

### 6. Whispers — Pool Model (`window.WHISPERS`)
Appended at the bottom of `<body>`. N independent forever-loops, each slot self-reschedules via `setTimeout`.

```js
window.WHISPERS = {
  enabled, color,            // hex string; CSS var --whisp-color set on .hero
  poolSize, minDist,         // slots count; min px between active elements
  gapMin, gapMax,            // seconds between cycles per slot
  // ghost type (dim, large, slow fade)
  ghostOpMin/Max, ghostHoldMin/Max, ghostSzMin/Max,
  // flash type (bright, smaller, fast)
  flashOpMin/Max, flashHoldMin/Max, flashSzMin/Max,
  rotMin, rotMax,            // rotation degrees
  edgePad,                   // px inset from hero edges
  edges: [left, right, top, bottom],  // which edges spawn from
  setColor(hex),             // updates --whisp-color CSS var on .hero
  setPoolSize(n)             // grow/shrink pool; new slots get random initial delay
}
```

**Spawn logic:** random edge → random position along that edge → retry up to 10× if `isTooClose()` (checks all live `.whisper` elements) → create `<div class="whisper">` → append to `.hero` → fadeIn → hold → fadeOut → remove.

**CSS custom property `--wr`** set inline per element for per-element rotation inside `@keyframes whisp-shiver` (avoids transform override conflict).

**Active slots tracked in `Set activeSlots`**. Slot retires if `idx >= poolSize`.

---

## Config Keys (DEFAULTS)
```js
// Title jitter
amp, spd

// Glow / breathe
glow, glowInv, breatheOn, breatheSpeed, breatheDepth

// Per-word disable
wordDisabled: [bool×4]  // index = word index

// Dust
dustOn, dustCount, dustSpeed, dustOpacity, dustSize, dustParallax

// Fog
fogOn, fogOpacity, fogHeight, fogSpeed, fogColor,
fogScale, fogWarp, fogFlowX, fogFlowY, fogOctaves, fogThrottle

// Scratch grain
scratchTex   // overlay opacity 0–1

// Whispers
whispOn, whispColor,
whispPoolSize, whispMinDist,
whispGapMin, whispGapMax,
whispGOpMin, whispGOpMax, whispGHoldMin, whispGHoldMax, whispGSzMin, whispGSzMax,
whispFOpMin, whispFOpMax, whispFHoldMin, whispFHoldMax, whispFSzMin, whispFSzMax,
whispRotMin, whispRotMax,
whispEdgePad, whispEdges: [bool×4]
```

---

## Dev Panel DOM IDs (control → config key mapping)
| ID | Config key |
|---|---|
| `amp-slider` | amp |
| `spd-slider` | spd |
| `glow-slider` | glow |
| `glow-invert` | glowInv |
| `breathe-on` | breatheOn |
| `bspd-slider` | breatheSpeed |
| `bdep-slider` | breatheDepth |
| `scratch-tex` | scratchTex |
| `dust-on/count/speed/opacity/size/parallax` | dust* |
| `fog-on/opacity/height/speed/color/scale/warp/flowx/flowy/oct/throttle` | fog* |
| `whisp-on/color/pool/mindist` | whisp* |
| `whisp-gap-min/max` | whispGapMin/Max |
| `whisp-gop/ghold/gsz -min/-max` | whispGOp/Hold/Sz Min/Max |
| `whisp-fop/fhold/fsz -min/-max` | whispFOp/Hold/Sz Min/Max |
| `whisp-rot-min/max` | whispRotMin/Max |
| `whisp-pad` | whispEdgePad |
| `whisp-edge-left/right/top/bottom` | whispEdges[0–3] |

---

## Key Patterns / Conventions
- **Adding a new effect param:** add to DEFAULTS, add control to relevant dev pane, read in `applyConfig`, write in `getConfig`, wire listener calling the live system.
- **`mergeWithDefaults`** must be called on any loaded config — handles stale/partial JSON gracefully. Arrays (`wordDisabled`, `whispEdges`) need explicit `.slice()` copy, done there.
- **Slider display values** follow pattern: `<div class="dev-row"><span>Label</span><span id="X-val">default</span></div>` + `<input id="X">`. Listener updates both the display span and the live system.
- **Dev minmax inputs** (whisper controls): `.dev-minmax` flex row, `.dev-minmax-label` above. Both min/max inputs share a single `dev-minmax` div.
- **`dynStyle()`** returns a `<style>` element for injecting runtime CSS (used by title freeze/unfreeze).
- **Hero image:** add `background-image: url('hero.jpg')` to `.hero-bg`; comment in HTML explains this.

---

## Known Constraints
- GLSL ES 1.0 (WebGL 1) — no dynamic loop bounds; fog octave loops are manually unrolled up to 5.
- Fog canvas renders at 0.5× resolution — intentional for performance.
- Whisper pool grow works; shrink relies on slots self-retiring when `idx >= poolSize` (eventual).
- `window.dustReinit()` must be called explicitly after changing count or speed — not automatic.
- FSA file handle stored in IndexedDB key `"cfgHandle"`. If site is moved/reloaded without re-linking, falls back to localStorage.
