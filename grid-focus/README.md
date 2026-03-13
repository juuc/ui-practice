# Grid Focus: Scrollable Bubble Grid with Distance-Based Focus Effect

A single-file canvas-based UI implementing a scrollable grid of emotion-state bubbles inspired by Apple Watch app layouts and mobile mood check-in interfaces. Features distance-based focus scaling, organic shape morphing, deterministic overlap resolution, and toroidal infinite scroll.

## Live Demo

Open `index.html` in a modern browser. No build step required.

## Technical Approach

### Grid Layout System

The foundation is a square grid with deterministic spacing:

- **Base cell size**: 2 × BASE_R + GAP (128px default)
- **STRIDE**: Controls spacing between bubble centers
- **Grid dimensions**: 13 cols × 15 rows (195 bubbles)

Each bubble has a `homeX` and `homeY` representing its grid position. The layout engine computes target positions from home positions, ensuring cells never "drift" across frame boundaries—only arrangement changes.

### Focus Effect: Distance-Based Scaling with Smoothstep Easing

The scaling effect is purely distance-driven:

```
distance = hypot(cell_x - viewport_center_x, cell_y - viewport_center_y)

if distance < FOCUS_RADIUS:
  targetScale = 1 + (MAX_SCALE - 1) × smoothstep(1 - distance / FOCUS_RADIUS)
else:
  targetScale = 1
```

**Smoothstep function** provides natural easing (cubic Hermite):

```javascript
smoothstep(t) = t × t × (3 - 2t)  // for t ∈ [0, 1]
```

This creates a smooth falloff from 2× scale at center to 1× at FOCUS_RADIUS (320px). Actual render scale is lerped each frame to avoid jitter:

```
scale += (targetScale - scale) × SCALE_LERP  // SCALE_LERP = 0.06
```

### Organic Shape Morphing

Each bubble is assigned a shape function that modulates radius in polar coordinates:

```javascript
modR(θ) = r × shapeFn(θ) / max(shapeFn)  // normalized to fit in r
```

10 predefined shapes:
- Squircle (soft square)
- 4/6/8-petal flowers
- Bowtie / hourglass
- Pinched oval
- Cross / plus
- Multi-petal variants

**Key insight**: Shapes are normalized so peaks equal the bubble radius and valleys recede inward. This keeps morphing contained within cell bounds—no overflow.

Morph state is time-based and fully decoupled from position:

```
if within MORPH_RADIUS (250px):
  targetMorph = smoothstep(1 - distance / MORPH_RADIUS)
  morphAmt += step × (targetMorph > morphAmt ? 1 : -1)
else:
  morphAmt = max(0, morphAmt - step)
```

Rendering interpolates between circle and organic shape:

```javascript
modR = r × (1 + (raw - 1) × morphAmt)  // circle when morphAmt=0, shape when morphAmt=1
```

**Duration**: 40 frames at 60fps = ~0.67 seconds for full morph.

### Separation of Concerns: Arrangement vs Transformation

This is the core architecture breakthrough:

| Concern | Responsibility | Properties |
|---------|---|---|
| **Arrangement** | Where cells are positioned (displacement resolution) | Deterministic, O(n²), physics-free |
| **Transformation** | How cells are visually rendered (shape, scale, rotation) | Time-based, contained within cell bounds, fully reversible |

**Arrangement** is computed fresh each frame from home positions. **Transformation** is applied purely in rendering without affecting the next frame's simulation.

This separation prevents:
- Morphing shapes causing position drift
- Overlap resolution fighting with visual effects
- Unpredictable accumulation of state

### Deterministic Displacement with Iterative Overlap Resolution

Layout computation happens in three steps:

**Step 1**: Compute target scale for each bubble (distance-based, described above).

**Step 2**: Resolve overlaps iteratively (3 passes):

1. Initialize all `targetX/targetY` to `homeX/homeY`
2. Loop 3 times:
   - For each pair of bubbles (i, j):
     - Compute separation vector (using wraparound distance)
     - If `distance < minDist` (accounting for scaled radii):
       - Displace both bubbles away from each other
       - Lighter bubble (smaller radius) moves more
       - Movement proportional to radius ratio

```javascript
const minDist = a.r + b.r + GAP × 0.4;
const overlap = minDist - dist;
const wA = b.r / (a.r + b.r);  // bubble a moves by b's weight
const wB = a.r / (a.r + b.r);  // bubble b moves by a's weight
a.targetX -= nx × overlap × wA;
b.targetX += nx × overlap × wB;
```

**Complexity**: O(n²) per pass × 3 passes = O(3n²). For 195 bubbles, ~114k comparisons per frame at 60fps is acceptable.

**Why 3 passes?** Two passes miss cascading overlaps. Three passes converge most conflicts.

**Step 3**: Lerp render positions toward targets (decoupled from collision logic):

```
px += (targetX - px) × POS_LERP  // POS_LERP = 0.08
```

This smooths movement without stalling convergence.

### Infinite Scroll: Toroidal Wrapping with Modular Arithmetic

The grid wraps seamlessly when scrolled past edges using modular arithmetic:

```javascript
wrapCoord(v, period, center) {
  return ((v - center) % period + period × 1.5) % period - period / 2 + center;
}

wrapDelta(d, period) {
  if (d > period / 2) return d - period;
  if (d < -period / 2) return d + period;
  return d;  // shortest distance accounting for wraparound
}
```

For each bubble:
1. Compute wrapped position in viewport-relative coordinates
2. Check distance to viewport center
3. Apply focus scaling and morphing based on wrapped distance

In rendering, each bubble is drawn at 9 positions (3×3 grid of wrapped copies):

```javascript
for (let wy = -1; wy <= 1; wy++) {
  for (let wx = -1; wx <= 1; wx++) {
    const sx = b.px + scrollX + wx * GRID_W;
    const sy = b.py + scrollY + wy * GRID_H;
    drawBubbleAt(b, sx, sy);
  }
}
```

Scrolling is normalized to stay within one grid period:

```javascript
scrollX = ((scrollX % GRID_W) + GRID_W) % GRID_W;
scrollY = ((scrollY % GRID_H) + GRID_H) % GRID_H;
```

This keeps the 3×3 wrap valid and prevents precision loss from large numbers.

### Canvas Rendering: Device Pixel Ratio, Z-Ordering, Glow Effects

**Device pixel ratio support**:

```javascript
const dpr = window.devicePixelRatio || 1;
canvas.width = w * dpr;
canvas.height = h * dpr;
ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
```

Ensures crisp rendering on high-DPI displays.

**Depth ordering** by sorting bubbles by scale (larger = higher z-index):

```javascript
const sorted = [...bubbles].sort((a, b) => a.scale - b.scale);
```

Smallest bubbles render first, largest last. No explicit z-index needed.

**Visual effects per bubble**:

1. **Base fill**: HSL color computed from grid position + noise
2. **Glow**: Radial shadow scaled by `(scale - 1) × 20`
3. **Highlight**: Radial gradient at top-left
4. **Inner shadow**: Linear gradient at bottom (depth cue)
5. **Selection ring**: Stroked shape (matched to morph state)
6. **Label**: Text scaled and color-adjusted by scale

Shapes are traced via polar coordinates:

```javascript
tracePolarPath(ctx, x, y, r, shapeIdx, morphAmt, rotation) {
  for (let i = 0; i <= SHAPE_POINTS; i++) {
    const θ = (i / SHAPE_POINTS) × Math.PI × 2;
    const raw = shapeFn(θ + rotation) / maxR;  // normalized
    const modR = r × (1 + (raw - 1) × morphAmt);  // morph blend
    ctx.lineTo(x + Math.cos(θ) × modR, y + Math.sin(θ) × modR);
  }
  ctx.closePath();
}
```

**Off-screen culling**: Bubbles outside viewport + margin are skipped:

```javascript
if (sx + b.r < -50 || sx - b.r > w + 50 || ...) continue;
```

### Momentum-Based Drag Scrolling

Scroll is driven by user drag and physics momentum:

**During drag**:
- Track pointer movement and time delta
- Compute velocity as exponential moving average: `vel = 0.4 × current + 0.6 × previous`
- Apply scrolling immediately

**After release**:
- Momentum continues with friction: `vel *= 0.93` per frame
- Velocity fades naturally over ~15 frames

**Click detection** (to differentiate tap from drag):

```javascript
const didDrag = Math.abs(pointerDown - pointerUp) > 6;  // 6px threshold
```

Only fire click/selection on taps, not drags.

## Config Constants

Control behavior by modifying these values:

| Constant | Default | Purpose |
|----------|---------|---------|
| `BASE_R` | 42 | Bubble radius in pixels |
| `GAP` | 20 | Spacing between bubbles |
| `STRIDE` | 128 | Cell center-to-center distance |
| `COLS`, `ROWS` | 13, 15 | Grid dimensions |
| `FOCUS_RADIUS` | 320 | Distance where scaling starts (px) |
| `MAX_SCALE` | 2.0 | Maximum scale at viewport center |
| `POS_LERP` | 0.08 | Position smoothing (0-1, lower = slower) |
| `SCALE_LERP` | 0.06 | Scale smoothing (0-1, lower = slower) |
| `MORPH_DURATION` | 40 | Frames for complete morph in/out |
| `MORPH_RADIUS` | 250 | Distance within which shapes morph (px) |
| `SHAPE_POINTS` | 72 | Resolution of polar shape tracing |

Increasing `FOCUS_RADIUS` widens the focus zone. Decreasing `MAX_SCALE` softens the effect. Tweaking `*_LERP` values changes responsiveness—higher = snappier, lower = smoother.

## Tech Stack

- **Language**: Plain JavaScript (ES6+)
- **Rendering**: HTML5 Canvas 2D
- **No frameworks**: No React, Vue, Three.js, or similar
- **No libraries**: All math and rendering is hand-written
- **UI framework**: CSS Grid + Flexbox for header/vignette overlays

One-file implementation (~550 lines including styles, config, and comments).

## Browser Support

Requires:
- Canvas 2D context with transforms
- `requestAnimationFrame` for animation loop
- `pointer` events (falls back from touch/mouse)
- Device pixel ratio awareness

Works on modern Chrome, Firefox, Safari, and Edge. Mobile browsers supported (iOS Safari, Android Chrome).

## Future Improvements

- Customizable emotion labels (API binding)
- Haptic feedback on selection (iOS)
- Accessibility improvements (ARIA labels for screen readers)
- WebGPU rendering for larger grids
- Server-side state persistence
