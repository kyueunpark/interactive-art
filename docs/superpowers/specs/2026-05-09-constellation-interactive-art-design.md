# Constellation — Interactive Art Design Spec

**Date**: 2026-05-09  
**Status**: Approved

---

## Overview

An interactive art page where a model photo serves as the background canvas, with seed-based scattered dots overlaid on top. Users click dots to connect them with glowing constellation lines, creating a personal star-map artwork. The result can be saved as a PNG.

---

## Core Concept

- Model photo fills the screen, darkened to ~60% opacity so dots and lines read clearly
- 50–100 glowing dots are scattered over the photo at load time, placement determined by a seed value
- User clicks any dot to select it (turns orange/highlighted), then clicks a second dot to draw a glowing line between them
- Repeating this builds up a constellation pattern
- Same seed always produces the same dot layout — different seeds produce entirely different compositions

---

## Interaction Flow

1. Page loads → photo fills screen → dots appear with subtle glow
2. User clicks dot A → dot turns orange (selected state)
3. User clicks dot B → line appears between A and B, both dots return to white
4. Continue clicking to add more connections
5. `↻ New` button → regenerates with a new random seed (clears all lines)
6. `⟵ Undo` button → removes the last drawn line
7. `💾 Save` button → downloads the current canvas as PNG named `constellation-{seed}.png`

---

## Visual Design

| Element | Spec |
|---|---|
| Background photo | Full-screen, `object-fit: cover`, dimmed with dark overlay |
| Dark overlay | `rgba(0, 0, 10, 0.45)` — preserves photo while making dots pop |
| Dots | Radius 5–7px, white fill, soft glow (`box-shadow` or Canvas `shadowBlur`) |
| Selected dot | Orange (#d97757) with stronger glow |
| Lines | 1.5px stroke, `rgba(200, 210, 255, 0.55)`, glow effect |
| UI overlay | Fixed bottom-right corner, dark semi-transparent pill |
| Font | Poppins or system monospace for seed number |

---

## UI Controls (minimal, bottom-right)

```
[ seed: 2847 ]  [ ↻ New ]  [ ⟵ Undo ]  [ 💾 Save ]
```

- Seed display is read-only (shows current seed)
- `↻ New` generates a random seed and resets
- `⟵ Undo` removes the last connection
- `💾 Save` exports PNG

---

## Technical Architecture

| Decision | Choice | Reason |
|---|---|---|
| Stack | Single HTML file | No build step, instant open in browser |
| Canvas | HTML5 Canvas API (vanilla) | No dependencies beyond the photo |
| Randomness | Seeded PRNG (mulberry32) | Reproducible dot placement from seed |
| Photo loading | `<img>` embedded or drag-and-drop | Works offline, no server needed |
| Export | `canvas.toDataURL('image/png')` | Built-in, no library needed |

### Seeded Dot Placement

```js
function mulberry32(seed) {
  return function() {
    seed |= 0; seed = seed + 0x6D2B79F5 | 0;
    // ... returns float 0–1
  }
}

function generateDots(seed, count = 70) {
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
};
```

---

## Photo Integration

- Default: one model photo embedded as a base64 data URL or local file reference
- Optional (v2): drag-and-drop a photo onto the canvas to replace it
- The photo renders to the canvas first, then dots and lines draw on top

---

## Out of Scope (this version)

- Multiple photo switching
- Named constellation labels
- Sharing via URL (would need a backend)
- Mobile touch optimization (desktop-first)

---

## File Output

```
interactive-art/
└── constellation.html   ← single self-contained file
```

---

## Success Criteria

- [ ] Photo fills screen, looks beautiful
- [ ] Dots appear correctly for a given seed (same seed = same layout every time)
- [ ] Click-to-connect works reliably
- [ ] Lines glow and look like constellations
- [ ] Undo removes the last line
- [ ] Save exports a clean PNG
- [ ] New seed button produces a fresh layout
