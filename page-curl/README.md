# Page Curl

Per-pixel page curl effect using Canvas 2D and the fold-line reflection model.

## Technique

### Fold-Line Reflection Model

The fold line is the **perpendicular bisector** of the segment from the page corner to the drag tip. Every pixel is classified by its signed distance to this line:

| Region | Condition | Rendering |
|--------|-----------|-----------|
| Flat page | `dist < 0`, reflection out of bounds | Sample texture at pixel position |
| Flap (backface) | `dist < 0`, reflection in bounds | Sample texture at reflected position, apply backface tint + shadow |
| Background | `dist > 0` | Paper lifted away, show dark surface |

### Why Reflection?

An earlier approach used origin-based cylinder mapping (`theta = asin(dist/R)`) from the Expo blog post. This broke when the curl direction was changed — the texture coordinates didn't match the flat page when the curl was released.

Reflection-based mapping guarantees correctness: at `dist = 0` (the fold line), the reflected coordinate equals the original coordinate. Content always matches perfectly when flattened.

### Shadow

Two effects create depth:

- **Crease shadow** — darkens pixels within 15px of the fold line, simulating the paper curving away
- **Cylinder gradient** — `0.5 + 0.5 * sin(t * π/2 + 0.2)` across the flap, simulating light on a curved surface

### GPU Shader Application

In React Native (Skia + Reanimated), this same per-pixel logic runs as a fragment shader on the GPU. The drag position is fed as a uniform via Reanimated shared values on the UI thread — no JS bridge overhead, 60fps.

## Structure

Single `index.html` containing:

1. **Hero** — Interactive page curl with edge-drag and snap-back animation
2. **Step 1** — Fold line & signed distance (region classification)
3. **Step 2** — Reflection-based texture mapping
4. **Step 3** — Shadow & depth (crease + cylinder)
5. **Step 4** — Backface rendering (grey-blue tint)
6. **Step 5** — All combined
