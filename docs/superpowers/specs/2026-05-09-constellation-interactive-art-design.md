# Constellation — Interactive Art Design Spec

**Date**: 2026-05-09  
**Status**: Approved

---

## Overview

An interactive art page where a user uploads a model photo as the background canvas, with seed-based scattered dots overlaid on top. Users click dots to connect them with orange constellation lines, creating a personal star-map artwork. The result can be saved as a PNG.

---

## Core Concept

- User uploads a photo via drag & drop → photo fills the screen at full brightness (no darkening)
- Up to 15 orange dots are scattered over the photo at load time, placement determined by a fixed seed
- User clicks any dot to select it (highlighted orange), then clicks a second dot to draw an orange line between them
- Repeating this builds up a constellation pattern
- Same seed always produces the same dot layout for a given photo dimensions

---

## Interaction Flow

1. **Photo upload** → drag & drop a photo onto the canvas → dots auto-placed (up to 15)
2. Click dot A → dot activates (highlighted)
3. Click dot B → orange line connects A and B, both stay orange
4. Continue clicking to complete the constellation
5. `💾 Save` button → downloads the current canvas as PNG

---

## Visual Design

| Element | Spec |
|---|---|
| Background photo | Full-screen, `object-fit: cover`, **no dark overlay** — full brightness |
| Dots | Radius 6–8px, **orange (#d97757)** fill, subtle glow |
| Selected dot | Brighter orange with stronger glow to indicate active state |
| Lines | 2px stroke, **orange (#d97757)**, subtle glow effect |
| UI overlay | Minimal, fixed bottom-right corner |

---

## UI Controls (minimal, bottom-right)

```
[ 📁 Upload Photo ]  [ 💾 Save PNG ]
```

- `📁 Upload Photo` — opens file picker (also supports drag & drop onto canvas)
- `💾 Save PNG` — exports the full canvas (photo + dots + lines) as PNG

---

## Technical Architecture

| Decision | Choice | Reason |
|---|---|---|
| Stack | Single HTML file | No build step, instant open in browser |
| Canvas | HTML5 Canvas API (vanilla) | No dependencies |
| Randomness | Seeded PRNG (mulberry32) | Reproducible dot placement |
| Photo loading | Drag & drop + file input | Works offline, no server needed |
| Export | `canvas.toDataURL('image/png')` | Built-in, no library needed |

### Seeded Dot Placement

```js
function mulberry32(seed) {
  return function() {
    seed |= 0; seed = seed + 0x6D2B79F5 | 0;
    // ... returns float 0–1
  }
}

function generateDots(seed, count = 15) {
  const rand = mulberry32(seed);
  return Array.from({ length: count }, () => ({
    x: rand() * canvas.width,
    y: rand() * canvas.height,
  }));
}
```

### State Model

```js
const state = {
  seed: 2847,
  dots: [],          // { x, y }[]
  connections: [],   // [dotIndexA, dotIndexB][]
  selected: null,    // dot index or null
  photo: null,       // ImageBitmap
};
```

---

## Photo Integration

- User drags a photo onto the canvas or clicks the upload button
- Photo renders to canvas at full brightness (no overlay)
- Dots and lines render on top of the photo

---

## Out of Scope (this version)

- Seed navigation / changing dot layout
- Undo (can be added later)
- Named constellation labels
- Mobile touch optimization (desktop-first)

---

## File Output

```
interactive-art/
└── constellation.html   ← single self-contained file
```

---

## Success Criteria

- [ ] Drag & drop photo upload works
- [ ] Photo fills screen at full brightness
- [ ] Up to 15 orange dots appear at consistent positions for a given seed
- [ ] Click-to-connect draws orange lines between dots
- [ ] Save exports a clean PNG with photo + constellation
