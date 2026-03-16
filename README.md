# UI Practice

Vanilla JS/CSS implementations of interactive UI effects. No frameworks, no dependencies — just raw browser APIs.

Each project is a self-contained `index.html` — no build step. Try them live via GitHub Pages or clone and open locally.

## Projects

| # | Project | Technique | Live Demo |
|---|---------|-----------|-----------|
| 01 | [grid-focus](./grid-focus/) | Canvas, polar shapes, iterative overlap resolution | [Try it](https://juuc.github.io/ui-practice/grid-focus/) |
| 02 | [card-swap](./card-swap/) | CSS 3D transforms, perspective, custom tween engine | [Try it](https://juuc.github.io/ui-practice/card-swap/) |
| 03 | [electric-border](./electric-border/) | Canvas, octaved value noise, path displacement | [Try it](https://juuc.github.io/ui-practice/electric-border/) |
| 04 | [page-curl](./page-curl/) | Canvas, fold-line reflection, per-pixel shading | [Try it](https://juuc.github.io/ui-practice/page-curl/) |

## Ad-hoc

Small exploratory demos that don't warrant their own project.

| Demo | What it shows | Live Demo |
|------|---------------|-----------|
| [axis-ball](./ad-hoc/axis-ball.html) | Interactive `translate3d` + `perspective` visualizer | [Try it](https://juuc.github.io/ui-practice/ad-hoc/axis-ball.html) |

## Running locally

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
