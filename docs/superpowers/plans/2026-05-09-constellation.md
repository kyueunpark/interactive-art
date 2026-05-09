# Constellation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a single-file interactive art page where users upload a model photo, see 15 orange dots overlaid on it, and click-connect them into constellation lines — then save as PNG.

**Architecture:** Single `constellation.html` file with vanilla Canvas API. State is a plain JS object. All rendering goes through one `render()` function that redraws photo → lines → dots on every state change.

**Tech Stack:** HTML5 Canvas API, vanilla JS (no libraries), CSS Flexbox for layout.

---

## File Structure

```
interactive-art/
└── constellation.html   ← the entire app lives here
```

Internal JS structure inside the `<script>` tag:

| Section | Responsibility |
|---|---|
| `state` object | Single source of truth: photo, dots, connections, selected dot |
| `mulberry32(seed)` | Seeded PRNG — pure function returning a float 0–1 |
| `generateDots()` | Populates `state.dots` using PRNG + current canvas dimensions |
| `render()` | Clears canvas → draws photo → draws lines → draws dots |
| `findDotAt(x, y)` | Hit-test: returns dot index within 20px radius, or null |
| Canvas click handler | Updates state.selected / state.connections, calls render() |
| Drag & drop handlers | Loads file → HTMLImageElement → resizes canvas → generateDots() → render() |
| Save button handler | canvas.toDataURL → download link click |

---

## Task 1: HTML Skeleton + CSS

**Files:**
- Create: `constellation.html`

- [ ] **Step 1: Create the file with full HTML structure**

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Constellation</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }

    body {
      background: #0a0a0f;
      width: 100vw;
      height: 100vh;
      display: flex;
      align-items: center;
      justify-content: center;
      overflow: hidden;
      font-family: 'Poppins', system-ui, sans-serif;
    }

    #canvas {
      display: block;
      max-width: 100vw;
      max-height: 100vh;
      cursor: crosshair;
    }

    /* Drop zone shown before photo is uploaded */
    #drop-zone {
      position: fixed;
      inset: 0;
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: center;
      color: #888;
      font-size: 18px;
      gap: 16px;
      pointer-events: none;
    }

    #drop-zone.hidden { display: none; }

    #drop-zone .icon { font-size: 48px; }

    /* UI controls — bottom right */
    #controls {
      position: fixed;
      bottom: 24px;
      right: 24px;
      display: flex;
      gap: 10px;
      align-items: center;
    }

    .btn {
      background: rgba(20, 20, 30, 0.85);
      border: 1px solid rgba(217, 119, 87, 0.4);
      color: #d97757;
      padding: 10px 18px;
      border-radius: 8px;
      font-size: 14px;
      font-weight: 500;
      cursor: pointer;
      transition: background 0.2s, border-color 0.2s;
      backdrop-filter: blur(8px);
    }

    .btn:hover {
      background: rgba(217, 119, 87, 0.15);
      border-color: #d97757;
    }

    /* hidden file input */
    #file-input { display: none; }

    /* drag-over highlight */
    body.drag-over #drop-zone {
      background: rgba(217, 119, 87, 0.05);
      pointer-events: auto;
    }
  </style>
</head>
<body>

  <canvas id="canvas" width="1200" height="800"></canvas>

  <div id="drop-zone">
    <div class="icon">📸</div>
    <div>사진을 여기에 드래그하거나 업로드하세요</div>
    <button class="btn" id="upload-btn">사진 선택</button>
  </div>

  <div id="controls">
    <button class="btn" id="upload-btn-2">📁 사진 바꾸기</button>
    <button class="btn" id="save-btn">💾 저장</button>
  </div>

  <input type="file" id="file-input" accept="image/*">

  <script>
    // JS goes here in subsequent tasks
  </script>
</body>
</html>
```

- [ ] **Step 2: Open in browser and verify layout**

Open `constellation.html` directly in a browser.  
Expected: dark background, centered drop-zone text with upload button, controls in bottom-right.

- [ ] **Step 3: Commit**

```bash
cd ~/Desktop/dev/interactive-art
git add constellation.html
git commit -m "feat: HTML skeleton and CSS layout"
```

---

## Task 2: Photo Upload (Drag & Drop + File Picker)

**Files:**
- Modify: `constellation.html` — replace the empty `<script>` block

- [ ] **Step 1: Add state object and photo loading logic**

Replace the `// JS goes here` comment with:

```js
// ─── State ───────────────────────────────────────────────────────────────────
const state = {
  seed: 42,
  photo: null,       // HTMLImageElement
  dots: [],          // [{x, y}]
  connections: [],   // [[indexA, indexB]]
  selected: null,    // dot index or null
};

// ─── Canvas refs ─────────────────────────────────────────────────────────────
const canvas = document.getElementById('canvas');
const ctx = canvas.getContext('2d');
const dropZone = document.getElementById('drop-zone');

// ─── Photo loading ────────────────────────────────────────────────────────────
function loadPhoto(file) {
  if (!file || !file.type.startsWith('image/')) return;

  const url = URL.createObjectURL(file);
  const img = new Image();

  img.onload = () => {
    state.photo = img;

    // Fit photo to viewport while preserving aspect ratio
    const maxW = window.innerWidth;
    const maxH = window.innerHeight;
    const ratio = Math.min(maxW / img.naturalWidth, maxH / img.naturalHeight);
    canvas.width  = Math.round(img.naturalWidth  * ratio);
    canvas.height = Math.round(img.naturalHeight * ratio);

    dropZone.classList.add('hidden');
    generateDots();
    render();
    URL.revokeObjectURL(url);
  };

  img.src = url;
}

// ─── Drag & drop ─────────────────────────────────────────────────────────────
document.addEventListener('dragover', (e) => {
  e.preventDefault();
  document.body.classList.add('drag-over');
});

document.addEventListener('dragleave', (e) => {
  if (e.relatedTarget === null) document.body.classList.remove('drag-over');
});

document.addEventListener('drop', (e) => {
  e.preventDefault();
  document.body.classList.remove('drag-over');
  loadPhoto(e.dataTransfer.files[0]);
});

// ─── File input ───────────────────────────────────────────────────────────────
const fileInput = document.getElementById('file-input');
fileInput.addEventListener('change', () => loadPhoto(fileInput.files[0]));

document.getElementById('upload-btn').addEventListener('click',  () => fileInput.click());
document.getElementById('upload-btn-2').addEventListener('click', () => fileInput.click());

// ─── Render (stub — draws photo only for now) ─────────────────────────────────
function generateDots() {}  // stub

function render() {
  ctx.clearRect(0, 0, canvas.width, canvas.height);
  if (state.photo) {
    ctx.drawImage(state.photo, 0, 0, canvas.width, canvas.height);
  }
}
```

- [ ] **Step 2: Verify photo upload works**

Open `constellation.html`, drag a photo onto the page (or click "사진 선택").  
Expected: photo fills the canvas at full brightness, drop-zone disappears.

- [ ] **Step 3: Commit**

```bash
git add constellation.html
git commit -m "feat: photo upload via drag-and-drop and file picker"
```

---

## Task 3: Seeded Dot Placement

**Files:**
- Modify: `constellation.html` — replace the `generateDots()` stub

- [ ] **Step 1: Add mulberry32 PRNG above the state object**

Add this function before the `// ─── State` comment:

```js
// ─── Seeded PRNG (mulberry32) ─────────────────────────────────────────────────
function mulberry32(seed) {
  return function () {
    seed |= 0;
    seed = (seed + 0x6D2B79F5) | 0;
    let t = Math.imul(seed ^ (seed >>> 15), 1 | seed);
    t = (t + Math.imul(t ^ (t >>> 7), 61 | t)) ^ t;
    return ((t ^ (t >>> 14)) >>> 0) / 4294967296;
  };
}
```

- [ ] **Step 2: Replace the `generateDots()` stub with the real implementation**

```js
function generateDots() {
  const rand = mulberry32(state.seed);
  const pad = 50;  // keep dots away from edges
  state.dots = Array.from({ length: 15 }, () => ({
    x: pad + rand() * (canvas.width  - pad * 2),
    y: pad + rand() * (canvas.height - pad * 2),
  }));
}
```

- [ ] **Step 3: Update `render()` to draw dots after the photo**

Replace the `render()` function:

```js
function render() {
  ctx.clearRect(0, 0, canvas.width, canvas.height);

  // 1. Photo
  if (state.photo) {
    ctx.drawImage(state.photo, 0, 0, canvas.width, canvas.height);
  } else {
    ctx.fillStyle = '#0a0a0f';
    ctx.fillRect(0, 0, canvas.width, canvas.height);
  }

  // 2. Lines (drawn before dots so dots appear on top)
  // — added in Task 4

  // 3. Dots
  for (let i = 0; i < state.dots.length; i++) {
    const { x, y } = state.dots[i];
    const isSelected = state.selected === i;

    ctx.beginPath();
    ctx.arc(x, y, isSelected ? 9 : 7, 0, Math.PI * 2);
    ctx.fillStyle = '#d97757';
    ctx.shadowColor = isSelected ? '#ff9944' : '#d97757';
    ctx.shadowBlur = isSelected ? 24 : 14;
    ctx.fill();
  }

  ctx.shadowBlur = 0; // reset shadow so it doesn't bleed elsewhere
}
```

- [ ] **Step 4: Verify dots appear on photo**

Upload a photo.  
Expected: 15 orange glowing dots scattered over the photo. Same positions every time for the same photo size (seed is fixed at 42).

- [ ] **Step 5: Commit**

```bash
git add constellation.html
git commit -m "feat: seeded dot placement with orange glow"
```

---

## Task 4: Click-to-Select and Click-to-Connect

**Files:**
- Modify: `constellation.html` — add hit-test + click handler, update `render()` to draw lines

- [ ] **Step 1: Add `findDotAt()` helper**

Add this function after `generateDots()`:

```js
function findDotAt(x, y) {
  const HIT = 22; // hit radius in canvas pixels — slightly larger than dot
  for (let i = 0; i < state.dots.length; i++) {
    const dx = state.dots[i].x - x;
    const dy = state.dots[i].y - y;
    if (dx * dx + dy * dy <= HIT * HIT) return i;
  }
  return null;
}
```

- [ ] **Step 2: Add canvas click handler**

Add this after the `fileInput` event listeners:

```js
canvas.addEventListener('click', (e) => {
  if (!state.photo) return; // ignore clicks before photo is loaded

  // Convert screen coords → canvas coords (handles CSS scaling)
  const rect = canvas.getBoundingClientRect();
  const scaleX = canvas.width  / rect.width;
  const scaleY = canvas.height / rect.height;
  const x = (e.clientX - rect.left) * scaleX;
  const y = (e.clientY - rect.top)  * scaleY;

  const hit = findDotAt(x, y);

  if (hit === null) {
    // Clicked empty space — deselect
    state.selected = null;
  } else if (state.selected === null) {
    // No dot selected yet — select this one
    state.selected = hit;
  } else if (state.selected === hit) {
    // Clicked same dot — deselect
    state.selected = null;
  } else {
    // Second dot clicked — add connection
    state.connections.push([state.selected, hit]);
    state.selected = null;
  }

  render();
});
```

- [ ] **Step 3: Add line drawing to `render()`**

Inside `render()`, replace the `// 2. Lines` comment with:

```js
  // 2. Lines
  if (state.connections.length > 0) {
    ctx.strokeStyle = '#d97757';
    ctx.lineWidth = 2;
    ctx.shadowColor = '#d97757';
    ctx.shadowBlur = 10;
    ctx.lineCap = 'round';

    for (const [i, j] of state.connections) {
      ctx.beginPath();
      ctx.moveTo(state.dots[i].x, state.dots[i].y);
      ctx.lineTo(state.dots[j].x, state.dots[j].y);
      ctx.stroke();
    }
  }
```

- [ ] **Step 4: Verify interaction**

Upload a photo.  
- Click dot → it glows brighter (selected)  
- Click another dot → orange line appears between them  
- Click the same dot twice → deselects  
- Click empty space → deselects  

- [ ] **Step 5: Commit**

```bash
git add constellation.html
git commit -m "feat: click-to-connect constellation lines"
```

---

## Task 5: PNG Export + Final Polish

**Files:**
- Modify: `constellation.html` — add save handler + drag-over visual feedback

- [ ] **Step 1: Add save button handler**

Add after the canvas click handler:

```js
document.getElementById('save-btn').addEventListener('click', () => {
  if (!state.photo) return;

  const link = document.createElement('a');
  link.download = `constellation-${state.seed}.png`;
  link.href = canvas.toDataURL('image/png');
  link.click();
});
```

- [ ] **Step 2: Add cursor feedback for dots**

Add a `mousemove` handler to switch cursor when hovering a dot:

```js
canvas.addEventListener('mousemove', (e) => {
  if (!state.photo) return;

  const rect = canvas.getBoundingClientRect();
  const x = (e.clientX - rect.left) * (canvas.width  / rect.width);
  const y = (e.clientY - rect.top)  * (canvas.height / rect.height);

  canvas.style.cursor = findDotAt(x, y) !== null ? 'pointer' : 'crosshair';
});
```

- [ ] **Step 3: Verify PNG export**

Upload a photo, draw a few constellation lines, click 💾 저장.  
Expected: PNG file downloaded named `constellation-42.png`, containing the photo with dots and lines.

- [ ] **Step 4: Final end-to-end check**

1. Open `constellation.html` fresh — dark background + drop zone shown  
2. Drag a model photo onto page — photo fills canvas, 15 dots appear  
3. Click dot A → glows brighter  
4. Click dot B → line connects them  
5. Add 4–5 more connections  
6. Click 💾 저장 → PNG downloaded with the full composition  
7. Click 📁 사진 바꾸기 → pick a new photo → dots reset to same pattern  

- [ ] **Step 5: Commit**

```bash
git add constellation.html
git commit -m "feat: PNG export and cursor hover feedback — Constellation v1 complete"
```

---

## Self-Review

**Spec coverage:**
- ✅ Photo upload (drag & drop) — Task 2
- ✅ Full brightness, no overlay — Task 2 (`ctx.drawImage` without overlay)
- ✅ Up to 15 orange dots, seeded — Task 3
- ✅ Click-to-select → click-to-connect, both dots stay orange — Task 4
- ✅ Orange lines — Task 4
- ✅ PNG save — Task 5
- ✅ Single HTML file — all tasks

**Placeholder scan:** No TBDs, no vague steps. All code blocks complete.

**Type consistency:**
- `state.dots` populated by `generateDots()`, read by `render()` and `findDotAt()` ✓
- `state.connections` populated by click handler as `[i, j]` pairs, iterated in `render()` as `[i, j]` ✓
- `state.selected` set/cleared by click handler, read by `render()` ✓
- `loadPhoto()` sets `state.photo`, read by `render()` ✓
