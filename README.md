# Modular Layout Builder

A browser-based configurator for modular light-bar systems. The customer draws a layout of hexagons, squares or octagons; the tool returns the complete bill of materials, the circuit plan, the overall dimensions, a 3D room preview and a request-for-quote summary. Every number is derived from a small set of product rules applied to the drawing, so any layout the tool allows is one that can be built and sold.

Prototype by Codup, v0.9. The engine was validated against a reference 19-hexagon layout (74 fixtures, 410 BOM line items) and reproduces it line for line.

## Contents

| File | Purpose |
| --- | --- |
| `engine.js` | Pure layout engine. No DOM, no dependencies. Turns cells into fixtures, connectors, circuits, polarity, inserts and the BOM. Portable to Node or a Next.js server route unchanged. |
| `view3d.js` | 3D room view built on three.js r128: fixtures, connectors, wall or ceiling, cables, light wash, orbit camera. |
| `app.js` | UI layer: SVG plan canvas, controls, legend toggles, stats, BOM table, RFQ summary, theme, persistence, 2D/3D switch. |
| `page.html` | Markup and CSS (design tokens, light and dark themes). No document skeleton; the build adds it. |
| `index.html` | Assembled page for publishing as a hosted artifact. Loads three.js from cdnjs. |
| `Modular_Layout_Builder.html` | Fully standalone build with three.js embedded. Open it from disk in any browser; works offline. |
| `build.sh` | Concatenates the sources into both HTML outputs and syntax-checks the scripts. |

## Running it

Open `Modular_Layout_Builder.html` in a browser. Nothing to install, no server.

To rebuild after editing a source file:

```bash
# three.js UMD build is expected at vendor/three.min.js (npm pack three@0.128.0 → package/build/three.min.js)
./build.sh
```

To use the engine on its own:

```js
require('./engine.js');
const E = globalThis.StripeEngine;   // global name kept for backward compatibility
const r = E.compute({
  shape: 'hexagon',          // 'hexagon' | 'square' | 'octagon'
  size: 36,                  // 18 | 24 | 36 | 48 (inches)
  finish: 'black',           // 'black' | 'white'
  mounting: 'surface',       // 'surface' | 'suspended'
  cells: E.SHAPES.hexagon.defaultCells,   // [[q, r], ...] lattice coordinates
});
r.fixtures;   // 74
r.lumens;     // 84508
r.watts;      // 1110
r.circuits;   // [{ index, fixtures, watts, feedEdge, colour }, ...]
r.bom;        // [{ section, item, desc, qty, note }, ...]
r.bounds;     // { w, h } in inches
```

`E.openSlots(shape, cells)` lists the empty neighbour positions a cell can be added to; `E.removable(shape, cells, cell)` says whether removing a cell keeps the layout connected.

## How the engine works

1. **Lattice → graph.** Each cell's corners are snapped to shared vertices and each side becomes an edge. A unique edge is one fixture; a unique vertex is one connector. Shared sides count once (19 hexagons → 74 fixtures, 56 connectors).
2. **Connector bodies.** Vertex degree decides the body: 2 → 2-way (120° hex / 90° square / 135° octagon), 3 → 3-way Y or T, 4 → 4-way X (square grids only), 1 → end cap (open runs only).
3. **Polarity and inserts.** Each fixture is oriented M→F by a depth-first traversal (tree edges away from the root, back edges toward the ancestor). Closed-cell layouts are bridgeless, so every connector receives at least one male and one female end. The gender pattern at each vertex (MF, MFF, MMF, MMFF…) selects the insert.
4. **Circuits.** Fixtures are ordered by a connected top-to-bottom sweep and cut into chunks of ≤ 600 W (40 × 15 W at 36"). Each cut is nudged up to 8 fixtures earlier if that moves the boundary onto 3-way junctions. Where circuits meet, a circuit-separator body replaces the normal body and isolates the minority leg.
5. **Dimensions.** Pitch = fixture length + 3.5" connector body; overall size is the corner-to-corner bounding box in inches and mm.
6. **BOM.** Fixtures, connector bodies, separators, power feeds (1 per circuit), inserts, mounting hardware (3 surface clips or 2 suspension kits per fixture).

## Reference layout (regression test)

19 hexagons, 36", black, surface mount:

| Metric | Expected |
| --- | --- |
| Fixtures / lumens / load | 74 / 84,508 lm / 1,110 W |
| Connector bodies | 20 × 2-way, 30 × 3-way, 6 circuit separators |
| Inserts | 20 M-F, 18 MFF, 18 MMF |
| Power feeds / mounting clips | 2 / 222 |
| Total line items | 410 |
| Length × height | 316.0 × 307.9 in (corner-to-corner bounding box) |
| Circuit split | 555 W / 555 W (balanced; a fill-first 600 / 510 split is equally valid) |

Run `node -e "require('./engine.js'); ..."` with the defaults to reproduce.

## Provisional data

These values are placeholders until confirmed against the manufacturer's product catalogue. They live at the top of `engine.js` and are the only things that should change when the catalogue is confirmed.

- Per-fixture watts and lumens: 36" = 15 W / 1,142 lm; 18", 24" and 48" scaled by length.
- Item codes marked `*` in the BOM: `BAR-CONN-2W/3W/4W`, `BAR-FEED`, `BAR-SURMCLIP`, `BAR-SUSPKIT`. The `BAR` family prefix is a placeholder; replace it with the real product family code.
- Insert item numbers: not yet assigned.
- Connector body length 3.5", 600 W per feed, 3 clips / 2 suspension kits per fixture.

## Known limits

- One shape, one size and one finish per layout, by design (connector bodies are angle-specific).
- Editing happens in the 2D plan; the 3D view is a visualiser.
- Downloads and printing are not available when the page runs as a hosted artifact; the RFQ is copied as text. A server-side PDF is the production path.
- The standalone file is about 660 KB because three.js is embedded.

## Production path

The engine is framework-free and should move server-side (a Next.js route or server action) so that the BOM, pricing and the RFQ PDF come from one source of truth, with the rules tables (sizes, codes, limits) loaded from the product database rather than code.
