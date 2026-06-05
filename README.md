# ✦ Air Draw — Gesture-Based Air Drawing with AI Auto-Correct

> Draw in the air with your finger. Your webcam tracks your hand in real-time, renders neon glowing strokes, and AI automatically corrects your handwriting into clean cursive text.

![Air Draw Demo](https://img.shields.io/badge/Built%20With-MediaPipe%20%2B%20Claude%20AI-ff6b6b?style=for-the-badge)
![License](https://img.shields.io/badge/License-MIT-6bcb77?style=for-the-badge)
![Platform](https://img.shields.io/badge/Platform-Browser%20(Chrome%2FEdge)-4d96ff?style=for-the-badge)

---

## What Is This?

Air Draw is a single-file browser app that turns your laptop webcam into a mid-air drawing canvas. Raise your index finger and write anything — letters, words, shapes — and watch them appear as glowing neon strokes on screen. When you're done writing a word, the built-in AI reads your handwriting and replaces it with a perfectly clean, glowing cursive version automatically.

No installation. No server. One HTML file. Open it in Chrome and draw.

---

## Features

- **Real-time finger tracking** using Google MediaPipe Hands (21 landmark model)
- **4-layer neon glow rendering** — ambient bloom + color halo + neon core + white filament
- **Fire particle system** at your fingertip (60fps, independent animation loop)
- **AI handwriting auto-correct** — Claude Vision reads your stroke and replaces it with clean Dancing Script cursive
- **4 hand gestures** — draw, pause, palm erase, idle
- **Adaptive smoothing** — pure raw tracking at fast speeds, EMA smoothing at slow speeds
- **RDP + Chaikin pipeline** — removes tremor noise, smooths curves before rendering
- **Camera quality controls** — brightness, contrast, saturation, sharpness, opacity
- **7 neon colors** + adjustable brush size
- **Save your drawing** as PNG with dark background

---

## Demo

| Gesture | Action |
|---|---|
| ☝️ Index finger up | Draw |
| ✌️ Index + middle up | Pause / lift pen |
| 🖐 All four fingers up | Palm erase (wipes area under hand) |
| ✊ Fist | Idle |

---

## How to Use

### 1. Download
Download `air-draw.html` from this repository (single file, ~1400 lines).

### 2. Open in Chrome or Edge
```
Double-click air-draw.html  →  Opens in browser
```
> Firefox is not recommended — MediaPipe performs best in Chromium-based browsers.

### 3. Allow Camera
Click **Allow** when the browser asks for camera permission. The app needs your webcam to track your hand.

### 4. Wait for Model to Load
A loading screen appears while MediaPipe downloads the hand-tracking model (~8MB, one-time). This takes 5–15 seconds depending on your connection.

### 5. Start Drawing
- Hold up your **index finger only** → drawing starts immediately
- Move your hand to write letters or draw shapes
- **Put two fingers up** to lift the pen (like picking up a pen mid-sentence)
- **Open your full palm** to erase the area under your hand
- After ~1 second of pen-lift, **AI auto-correct fires** and replaces your strokes with clean neon text

---

## Project Structure

```
air-draw/
│
├── air-draw.html        ← The entire app (HTML + CSS + JS, single file)
└── README.md            ← This file
```

Everything is self-contained. No `node_modules`, no build step, no backend.

---

## How It Works — Technical Deep Dive

### Hand Tracking

The app uses **Google MediaPipe Hands** loaded from CDN. Every video frame is passed to the model which returns 21 3D landmarks for the detected hand.

```
Landmark 8  = Index fingertip
Landmark 5  = Index knuckle
Landmark 9  = Palm center (base of middle finger)
```

To find the **exact visual tip** of the finger (not just the landmark center), the app extends 12% past landmark 8 along the finger direction vector:

```js
const OFFSET = 0.12;
rawX = tipX8 + (fdx / flen) * flen * OFFSET;
rawY = tipY8 + (fdy / flen) * flen * OFFSET;
```

Since the webcam feed is mirrored, all X coordinates are flipped: `x = (1 - lm.x) * canvasWidth`.

### Adaptive Smoothing

Raw landmark positions jump frame-to-frame. A fixed smoothing value creates lag at fast speeds. The app uses a 3-tier adaptive EMA (Exponential Moving Average):

```
Speed > 12px/frame  →  Pure raw position (zero lag, 1:1 with hand)
Speed 5–12px/frame  →  70/30 EMA blend (slight smoothing)
Speed < 5px/frame   →  50/50 EMA blend (steady for fine detail)
```

When switching back from fast mode, the EMA is resynced to raw position to prevent position jumps.

### Drawing Engine

The drawing pipeline has two stages:

**Stage 1 — Live Preview (while finger is down)**
As points come in, they are rendered using quadratic bezier curves through a rolling window of 3 points. This gives a smooth preview with no waiting.

**Stage 2 — Clean Final Render (when finger lifts)**
1. **RDP Simplification** (epsilon = 3px) — Ramer-Douglas-Peucker removes collinear redundant points and tremor noise while preserving letter shape
2. **Chaikin Subdivision** (3 iterations) — Repeatedly cuts corners, producing smooth curves with zero overshoot
3. The cleaned path replaces the live preview via full history redraw

### Neon Rendering

Each stroke is rendered as 4 stacked canvas paths drawn in a single `beginPath`:

| Layer | Width | Alpha | Effect |
|---|---|---|---|
| Ambient bloom | sz × 1.0 | 4% | Soft outer glow |
| Color halo | sz × 0.55 | 18% | Color body |
| Neon core | sz × 0.6 + shadow | 100% | Main tube |
| White filament | sz × 0.12 | 60% | Bright inner center |

### Fire Particle System

A `requestAnimationFrame` loop runs at 60fps completely independent of MediaPipe (which runs at ~30fps). This loop:

1. Spawns 4–7 particles per frame at the fingertip when drawing
2. Updates all particle physics (`vy -= 0.06` upward gravity, `vx *= 0.97` drag)
3. Clears `overlayCanvas` and redraws: hand skeleton → particles → fingertip glow dot

Particles inherit the hue of the current color ± 15°, fade out over ~25 frames, and rise upward like real fire.

### AI Handwriting Auto-Correct

This is the most advanced feature. Here's the complete flow:

```
1. You draw a word (one or more strokes)
2. You lift your finger (PAUSE gesture) for 900ms
3. App captures a bounding box around all pending strokes
4. Renders strokes onto a temporary canvas (dark bg, plain color lines)
5. Converts to base64 PNG
6. Sends to Claude API (claude-sonnet-4-20250514) with vision:
      "What letter(s) or word is written? Reply with ONLY the text."
7. Claude responds with recognized text (e.g. "love")
8. Rough strokes are removed from stroke history
9. Canvas is redrawn without them
10. A clean neon glyph is rendered using Dancing Script Bold cursive
11. The glyph is stored as type:'glyph' in history so it persists on resize/redraw
```

The AI fallback (if unrecognized or API unavailable) keeps the RDP+Chaikin smoothed version of the original strokes.

### Canvas Architecture

Three stacked full-viewport layers:

```
┌─────────────────────────────┐
│  overlayCanvas (top)        │  ← Cleared every rAF: skeleton, particles, glow dot
├─────────────────────────────┤
│  drawCanvas                 │  ← Persistent: stroke history (never auto-cleared)
├─────────────────────────────┤
│  <video> (bottom)           │  ← Mirrored webcam feed, CSS filtered
└─────────────────────────────┘
```

The key architectural decision: **only rAF clears overlayCanvas**, never the MediaPipe callback. This prevents flicker.

---

## Configuration

All configurable values are at the top of the script block:

| Variable | Default | Description |
|---|---|---|
| `MIN_SPACING` | `2px` | Minimum distance between recorded stroke points |
| `AI_PAUSE_MS` | `900ms` | Debounce delay before triggering AI correction |
| `MAX_PARTICLES` | `120` | Maximum simultaneous fire particles |
| `OFFSET` | `0.12` | Finger tip extension factor (12% past landmark 8) |
| `brushSize` | `3` | Default brush size (1–20) |

---

## Browser Requirements

| Browser | Support |
|---|---|
| Chrome 88+ | ✅ Full support |
| Edge 88+ | ✅ Full support |
| Firefox | ⚠️ May work but not recommended |
| Safari | ❌ MediaPipe not supported |
| Mobile | ❌ Requires laptop/desktop webcam |

---

## Privacy

- All hand tracking runs **100% locally** in your browser using WebAssembly. No hand data is sent anywhere.
- The only network request is to the **Anthropic Claude API** when AI auto-correct fires — it sends a small PNG image of your strokes (no video, no personal data).
- You can turn off AI auto-correct with the **✦ button** on the sidebar — then the app works fully offline after the initial MediaPipe model download.

---

## Credits

Built with:
- [Google MediaPipe Hands](https://google.github.io/mediapipe/solutions/hands.html) — real-time hand landmark detection
- [Anthropic Claude API](https://www.anthropic.com) — handwriting recognition via vision
- [Dancing Script](https://fonts.google.com/specimen/Dancing+Script) — Google Fonts cursive typeface
- Inspired by gesture drawing projects and neon sign aesthetics

---

## License

MIT — use it, modify it, build on it.

---

*Made with ✦ and a lot of finger pointing*
