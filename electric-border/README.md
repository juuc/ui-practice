# Electric Border: Noise-Displaced Animated Rounded Rectangle

A single-file canvas-based UI component that renders an animated, jittery border using Perlin-like value noise. The border appears to shimmer with electric energy by displacing each point along the perimeter away from its original position using multiple octaves of continuous noise. As time advances, the noise landscape shifts smoothly, creating an organic, living effect.

## Live Demo

Open `index.html` in a modern browser. No build step required.

## Technical Approach

### Intuition 1: Walk a Path, Then Displace Each Point

The electric border effect is not drawn directly. Instead:

1. Define a clean geometric shape (rounded rectangle)
2. Sample points uniformly along its entire perimeter (straights + arcs)
3. Push each point off its original position by a noise-derived offset

This two-step process separates shape definition from visual distortion:

```javascript
// Step 1: Get the clean point on the rounded rect perimeter
const pt = roundedRectPoint(progress, left, top, bw, bh, r);

// Step 2: Displace by noise
const nx = octavedNoise(...);  // x-direction offset
const ny = octavedNoise(...);  // y-direction offset

const dx = pt.x + nx * DISPLACEMENT;  // displaced x
const dy = pt.y + ny * DISPLACEMENT;  // displaced y

ctx.lineTo(dx, dy);  // draw at displaced position
```

The original rounded rectangle is never drawn—only its displaced vertices are connected with line segments.

### Intuition 2: Noise Must Be Continuous, Not Random

The key to organic movement is that noise returns **smoothly varying values**, not random garbage.

**How it works:**

The `noise2D(x, y)` function implements value noise via:

1. **Grid-based pseudorandom values**: For each integer grid cell, compute a deterministic pseudorandom number using a sine-based hash:

```javascript
function pseudoRandom(x) {
  return ((Math.sin(x * 12.9898) * 43758.5453) % 1 + 1) % 1;
}
```

2. **Interpolate between grid corners**: Given a point (x, y), find the four surrounding grid corners and interpolate their values using Hermite smoothstep:

```javascript
const fx = x - i;      // fractional x
const fy = y - j;      // fractional y

// Smoothstep interpolation (cubic Hermite curve)
const ux = fx * fx * (3 - 2 * fx);
const uy = fy * fy * (3 - 2 * fy);

// Bilinear interpolation with smoothed weights
return a * (1 - ux) * (1 - uy) + b * ux * (1 - uy) +
       c * (1 - ux) * uy + d * ux * uy - 0.5;
```

The **smoothstep** function is critical: `t² × (3 - 2t)` creates C¹ continuity (smooth derivatives). Nearby inputs produce nearby outputs, creating flowing curves instead of jagged zigzags.

The final **- 0.5** centers the noise output in the range [-0.5, +0.5], preventing directional bias (noise anchored to +0.5 would pull points more in one direction than another).

### Intuition 3: Octaves = Detail at Multiple Scales

A single layer of noise produces smooth undulations. Ten layers stacked at increasing frequency and decreasing amplitude create fractal-like organic complexity.

```javascript
function octavedNoise(x, octaves, lacunarity, gain, amp, freq, time, seed, flatness) {
  let y = 0;
  let a = amp;      // amplitude decreases
  let f = freq;     // frequency increases

  for (let i = 0; i < octaves; i++) {
    const oa = i === 0 ? a * flatness : a;
    y += oa * noise2D(f * x + seed * 100, time * f * 0.3);
    f *= lacunarity;   // typical: 1.6 (higher frequency)
    a *= gain;         // typical: 0.7 (lower amplitude)
  }
  return y;
}
```

**Parameters used in the render loop:**

| Parameter | Value | Effect |
|-----------|-------|--------|
| `OCTAVES` | 10 | 10 layers stacked |
| `LACUNARITY` | 1.6 | Each layer is 1.6× faster |
| `GAIN` | 0.7 | Each layer is 70% the amplitude of the previous |
| `FREQUENCY` | 10 | Base frequency (wavelength of largest waves) |
| `FLATNESS` | 0 | First octave at full strength (0 = no attenuation) |

**Visual result:**
- Layer 1 (largest): Slow, sweeping waves
- Layers 2–5: Medium-frequency undulation
- Layers 6–10: Fine-grained jitter and detail

Together they create the illusion of chaotic energy with no single dominant frequency—purely organic.

### Intuition 4: Time as the Second Noise Dimension

Noise is a 2D function: `noise2D(position, time)`. As time advances each frame, the entire noise landscape shifts smoothly.

```javascript
const nx = octavedNoise(progress * 8, OCTAVES, LACUNARITY, GAIN,
                        inst.chaos, FREQUENCY, inst.time, 0, FLATNESS);
                        //                                   ^^^^^^^^^ time!
```

Each frame, `inst.time` increments:

```javascript
inst.time += dt * inst.speed;  // dt = frame delta in seconds, speed = user control
```

This shifts the noise evaluation position through time-space. A fixed point on the perimeter (e.g., `progress = 0.5`) will sample different noise values as time marches forward:

- `t=0`: `noise2D(..., 0)` → some offset
- `t=0.016`: `noise2D(..., 0.016)` → slightly different offset
- `t=0.032`: `noise2D(..., 0.032)` → another variation

The smoothness of noise ensures the offset changes smoothly frame-to-frame, creating fluid animation rather than flicker.

### Intuition 5: Perimeter as 1D Parameter (t = 0→1)

A rounded rectangle is a complex 2D shape with 4 straight edges and 4 corner arcs. Rather than define it in 2D coordinates, parameterize it as a single 1D value `t ∈ [0, 1]` representing normalized progress along the entire perimeter.

The function `roundedRectPoint(t, left, top, w, h, r)` maps this 1D parameter to a 2D (x, y) point:

```javascript
function roundedRectPoint(t, left, top, w, h, r) {
  const sw = w - 2 * r;  // straight-edge width
  const sh = h - 2 * r;  // straight-edge height
  const ca = (Math.PI * r) / 2;  // corner arc length
  const perim = 2 * sw + 2 * sh + 4 * ca;  // total perimeter
  const d = t * perim;  // distance along perimeter (0 to perim)

  let acc = 0;

  // Top edge
  if (d <= acc + sw) {
    const p = (d - acc) / sw;  // progress along this edge [0, 1]
    return { x: left + r + p * sw, y: top };
  }
  acc += sw;

  // Top-right corner
  if (d <= acc + ca) {
    return cornerPoint(left + w - r, top + r, r, -Math.PI / 2, Math.PI / 2, (d - acc) / ca);
  }
  acc += ca;

  // ... repeat for right edge, bottom-right corner, bottom edge, ...
}
```

**Benefit**: The entire perimeter is now 1D input space. Noise can be evaluated along this single parameter:

```javascript
const progress = i / samples;  // 0 to 1
const pt = roundedRectPoint(progress, ...);
const offset = octavedNoise(progress * 8, ...);  // 1D → scalar
```

This elegantly converts a 2D geometric problem into a 1D signal-processing problem.

### Intuition 6: Two Independent Noise Channels for x and y

Without this, all points would move only diagonally (x and y displacement would be correlated). Instead, use two independent noise samples with different seeds:

```javascript
const nx = octavedNoise(progress * 8, OCTAVES, LACUNARITY, GAIN,
                        inst.chaos, FREQUENCY, inst.time, 0, FLATNESS);  // seed=0
const ny = octavedNoise(progress * 8, OCTAVES, LACUNARITY, GAIN,
                        inst.chaos, FREQUENCY, inst.time, 1, FLATNESS);  // seed=1

const dx = pt.x + nx * DISPLACEMENT;
const dy = pt.y + ny * DISPLACEMENT;
```

The seed parameter (0 vs 1) shifts the pseudorandom hash input:

```javascript
const a = pseudoRandom(i + j * 57);           // for nx (seed=0)
const a = pseudoRandom(i + 100 + j * 57);    // for ny (seed=1)
```

Different seeds → different grid of random values → independent noise → points can move in any direction, creating turbulent, chaotic distortion.

### Intuition 7: Glow Sells the Effect

A pure canvas stroke without glow is just a squiggly line. The **glow effect** simulates light emission and is essential to selling the "electric" aesthetic.

**Canvas shadow/blur:**

```javascript
ctx.shadowColor = inst.color;
ctx.shadowBlur = 15;
ctx.stroke();
```

The `shadowBlur` property creates a Gaussian blur halo around the stroke, making it appear to glow.

**CSS glow layers** (independent from canvas, for extra luminosity):

```css
.eb-glow-1 {
  position: absolute;
  inset: -2px;
  border: 2px solid var(--electric-color);
  opacity: 0.15;
  filter: blur(8px);
}

.eb-glow-2 {
  position: absolute;
  inset: -4px;
  border: 2px solid var(--electric-color);
  opacity: 0.08;
  filter: blur(20px);
}

.eb-background-glow {
  background: radial-gradient(ellipse at center, var(--electric-color), transparent 70%);
  opacity: 0.04;
}
```

These layered blurred borders (8px and 20px blur) + background radial gradient create depth and sell the illusion of emitted light.

## Path Parameterization: Rounded Rectangle Edge + Corner Logic

The rounded rectangle is built by walking four edges (top, right, bottom, left) with corner arcs (top-right, bottom-right, bottom-left, top-left) interpolated at the junction points.

**Edge segments:**

- **Top**: Straight from (left + r, top) to (left + w - r, top)
- **Top-right corner**: Arc centered at (left + w - r, top + r), from -90° to 0°
- **Right**: Straight from (left + w, top + r) to (left + w, top + h - r)
- And so on...

**Corner arc:** Parameterized as angle sweep around a center point:

```javascript
function cornerPoint(cx, cy, r, startAngle, arcLen, progress) {
  const angle = startAngle + progress * arcLen;
  return { x: cx + r * Math.cos(angle), y: cy + r * Math.sin(angle) };
}
```

This keeps all four corners smooth and properly scaled.

## Render Loop and Animation

The animation loop runs once per frame via `requestAnimationFrame`:

```javascript
function draw(now) {
  const dt = (now - lastTime) / 1000;  // frame delta in seconds
  lastTime = now;

  for (const inst of instances) {
    inst.time += dt * inst.speed;  // advance time

    // Clear and scale to device pixel ratio
    ctx.clearRect(0, 0, canvas.width, canvas.height);
    ctx.scale(dpr, dpr);

    // Configure stroke and shadow
    ctx.strokeStyle = inst.color;
    ctx.shadowColor = inst.color;
    ctx.shadowBlur = 15;

    // Sample perimeter at regular intervals
    const samples = Math.floor(perim / 2);  // ~2px spacing

    ctx.beginPath();
    for (let i = 0; i <= samples; i++) {
      const progress = i / samples;  // 0 to 1
      const pt = roundedRectPoint(progress, left, top, bw, bh, r);

      // Displace by noise
      const nx = octavedNoise(progress * 8, ...inst.time...);
      const ny = octavedNoise(progress * 8, ...inst.time...);

      const dx = pt.x + nx * DISPLACEMENT;
      const dy = pt.y + ny * DISPLACEMENT;

      if (i === 0) ctx.moveTo(dx, dy);
      else ctx.lineTo(dx, dy);
    }
    ctx.closePath();
    ctx.stroke();  // applies shadow blur
  }

  requestAnimationFrame(draw);
}
```

**Key details:**

- **Sample spacing**: ~2px (perimeter / 2). Too sparse → visible linear segments. Too dense → wasted samples.
- **Device pixel ratio**: Canvas is rendered at `dpr × dpr` to ensure crisp rendering on high-DPI displays.
- **ResizeObserver**: When the container size changes, canvas dimensions are updated automatically.

## Configurable Parameters

Control the effect's character via data attributes:

```html
<div class="electric-border"
     data-electric
     data-color="#5227FF"
     data-chaos="0.12"
     data-speed="1"
     style="border-radius:24px">
```

| Parameter | Range | Effect |
|-----------|-------|--------|
| `data-color` | Any CSS color | Stroke and glow color |
| `data-chaos` | 0.01 to 0.5 | Amplitude of displacement (higher = more jittery) |
| `data-speed` | 0.1 to 4.0 | Time advancement multiplier (higher = faster animation) |
| `border-radius` | 0 to 60px | Corner radius of the base shape |

**Render-loop constants** (modify in `<script>`):

| Constant | Default | Purpose |
|----------|---------|---------|
| `OCTAVES` | 10 | Number of noise layers |
| `LACUNARITY` | 1.6 | Frequency multiplier per octave |
| `GAIN` | 0.7 | Amplitude multiplier per octave |
| `FREQUENCY` | 10 | Base frequency (larger = tighter waves) |
| `FLATNESS` | 0 | Attenuation of first octave (0 = full strength) |
| `DISPLACEMENT` | 60 | Maximum pixel offset per point |
| `BORDER_OFFSET` | 60 | Padding around the border for glow (px) |

Increase `CHAOS` for more aggressive jitter. Decrease `SPEED` for slower drift. Adjust `DISPLACEMENT` to control the maximum deviation from the true shape.

## Tech Stack

- **Language**: Plain JavaScript (ES6+)
- **Rendering**: HTML5 Canvas 2D
- **No frameworks**: No React, Vue, Three.js, or similar
- **No libraries**: All noise generation, path parameterization, and rendering is hand-written
- **Styling**: CSS custom properties for color theming

Single-file implementation (~410 lines including styles, config, and comments).

## Browser Support

Requires:
- Canvas 2D context with transforms and shadow effects
- `requestAnimationFrame` for animation loop
- `ResizeObserver` for responsive sizing
- Device pixel ratio awareness
- CSS Grid, Flexbox, and `color-mix()` for layout/theming

Works on modern Chrome, Firefox, Safari, and Edge (all versions from 2020+). Mobile browsers supported (iOS Safari 14+, Android Chrome 90+).

## Device Pixel Ratio Handling

High-DPI displays (e.g., Retina) require rendering at 2× resolution internally:

```javascript
const dpr = Math.min(devicePixelRatio || 1, 2);  // cap at 2x
canvas.width = w * dpr;   // internal resolution
canvas.height = h * dpr;
canvas.style.width = w + 'px';   // CSS resolution (normal)
canvas.style.height = h + 'px';
ctx.scale(dpr, dpr);  // scale graphics context
```

This ensures the stroke and shadow are crisp on high-DPI screens without bloating memory usage beyond 2×.

## Future Enhancements

- Customizable noise functions (Simplex, Perlin, cellular)
- Animation playback controls (pause/play)
- Mouse-following perturbation (displacement increases near cursor)
- Multiple geometric shapes (polygons, stars, custom SVG paths)
- Audio reactivity (displacement amplitude responds to audio frequencies)
- WebGL rendering for multiple borders or particle effects
