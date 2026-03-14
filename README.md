# UI Practice

Vanilla JS/CSS implementations of interactive UI effects. No frameworks, no dependencies — just raw browser APIs.

Each project is a self-contained `index.html` that can be opened directly or served locally.

## Projects

| # | Project | Technique | Demo |
|---|---------|-----------|------|
| 01 | [grid-focus](./grid-focus/) | Canvas, polar shapes, iterative overlap resolution | Scrollable bubble grid with organic shape morphing |
| 02 | [card-swap](./card-swap/) | CSS 3D transforms, perspective, custom tween engine | Card stack with elastic swap animation |
| 03 | [electric-border](./electric-border/) | Canvas, octaved value noise, path displacement | Noise-displaced animated border with glow |

## Ad-hoc

Small exploratory demos that don't warrant their own project.

| Demo | What it shows |
|------|---------------|
| [axis-ball](./ad-hoc/axis-ball.html) | Interactive `translate3d` + `perspective` visualizer |

## Running

```bash
cd <project>
python3 -m http.server 8000
# open http://localhost:8000
```

## Adding a new project

1. Create a directory: `my-effect/`
2. Put everything in `index.html` (single-file preferred)
3. Add a `README.md` if the technique is worth documenting
4. Add a row to the table above
