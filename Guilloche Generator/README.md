# Watch Dial Guilloché Generator

A single-file browser tool for designing engine-turned patterns on a watch dial, simulating
how they will look cut into metal, and exporting them as SVG, DXF or G-code.

The defining rule: **a pattern can never cross the dial's outline.** Every generated path is
clipped to the loaded outline — including any holes in it — before it is drawn or exported.

## Getting started

Open `guilloche-generator.html` in a browser. There is nothing to install and no build step.

It does need an internet connection the first time, because three.js is loaded from a CDN.
Offline, the 3D view will be blank.

A 34 mm sample dial with a centre hole and a date window loads automatically, so there is
something to work with straight away.

## The workflow

1. **Load a dial outline.** Open the **Dial** panel and click *Load Dial Outline (DXF)*, or
   drag a `.dxf` file anywhere onto the window. *Use Sample Dial* brings back the built-in one.
2. **Check it closed.** The panel reports how many closed loops were found. Open chains are
   drawn in red and are ignored — raise *Join Tol.* if a drawing has small gaps at its corners.
3. **Add pattern layers.** Each layer is one pattern with its own cutter and colour. Click a
   layer to edit it; click its colour swatch to recolour it.
4. **Set the cutter.** Choose a V/chamfer mill or a ball nose and a tip depth. The cut width
   follows from those two and is shown beneath them.
5. **Export.** SVG and DXF are one click. G-code opens a dialog for the machining setup.

### Reading the DXF

Lines, arcs, circles, ellipses, polylines (including bulges), splines and nested blocks are
all understood. Units come from the file's `$INSUNITS` header, with a manual override.

Only closed loops can contain a pattern. The largest becomes the dial; everything inside it
becomes a hole, so centre holes, date windows and subdials are cut around automatically.

## Patterns

| Pattern | What it is |
| --- | --- |
| Rosette Spiral | One unbroken spiral cut while the rosette rocks — classic rose engine work |
| Rosette Rings | The same cut as separate closed rings; a phase step between them gives the weave |
| Spirograph | A hypotrochoid — the flower a pen traces from inside a rolling circle |
| Spirograph Field | A family of those, nested along the pen offset until they weave; inside or outside |
| Spiral Vortex | Archimedean spirals turned into a family, leading the eye centre to rim |
| Straight-Line Waves | Parallel passes with the rubber rocking |
| Sunburst | Radial rays; add twist for a snailed dial |
| Concentric Circles | Plain turned circles; offset the centre for côtes circulaires |
| Cross Hatch | Families of parallel lines laid over each other |
| Clous de Paris | The square grid of tiny pyramids |
| Basket Weave | A checkerboard of hatched blocks |
| Barleycorn Arcs | Rings of overlapping arcs — grain d'orge |
| Trace Outline | Follows the dial outline itself, at an offset |

Every layer also has a **centre**, **rotation** and a **radius band**, so several can share
the dial — a sunburst in the middle, a rosette outside it, a traced border at the rim.

*Trace Outline* is the exception: it follows the outline itself and ignores both placement
and the edge margin, since at no offset it *is* the edge.

## Containment

**Edge Margin** (at the foot of the Dial panel) is how far short of the outline every path
stops. It applies to the outline and to every hole.

One thing to know: the margin clips the path the cutter *follows* — its centreline. The cut
itself is as wide as the tool makes it, so its edges stand half a width outside that. If the
margin is narrower than the widest cut on the dial, a hint appears telling you what to raise
it to.

## Simulating the cut

The **Show** panel over the top-left of the view controls what is drawn:

- **Dial** — the dial body
- **Cut** — the engraved relief, simulated as real recessed geometry
- **Outline** — the loaded loops (green closed, red open)
- **Paths** — the raw toolpaths, in each layer's colour
- **Grid**

The relief is genuinely recessed: the dial face is punched out along every cut so the walls
below show through. A chamfer mill leaves a V; a ball nose leaves an arc; both end in the
tool's own shape where it enters and leaves. Exaggerate **Tip Depth** to read a pattern on
screen — real guilloché is only a few hundredths of a millimetre deep.

Orbit with the left mouse button, pan with the right, zoom with the wheel. The cube in the
bottom-right corner snaps to standard views.

## Exporting

**SVG** and **DXF** write the clipped paths and the outline, re-origined on the dial centre.

**G-code** opens a dialog with a row per layer:

| Column | |
| --- | --- |
| Tool | Tool number. Layers sharing one are cut without a change between them |
| Safe Z | Retract height between paths |
| Feed / Plunge | Cutting and plunging feed rates |
| Speed | Spindle rpm — **0 cuts that layer with the spindle stopped** |

Below the table: **arc tolerance** (curves become G2/G3 arcs; 0 writes straight moves only),
**work offset**, **coolant**, and an **origin** offset in X/Y/Z if your part zero is not the
centre of the dial face.

The output targets **Fanuc, Haas and LinuxCNC** — a full safety line, `G43` tool length
offsets, `G64` blending, `G30` park and `M1` between tools. A GRBL-class controller will not
accept `G43 H` or `G64`.

## Saving your work

The two icons beside the title save and load the whole design as a `.json` file — every
layer with its pattern, cutter and machining settings, the dial settings, and the outline
itself. A saved file opens back to exactly what you had. You can also drop a saved `.json`
onto the window.

Nothing is saved automatically. Refreshing the page loses unsaved work.

## Notes and limits

- A DXF with no closed loop has no inside, so nothing can be contained and nothing is drawn.
- The simulation draws the exact toolpath; the exported G-code fits arcs to it, so the two
  differ by up to the arc tolerance.
- Very dense patterns are capped. If the cut simulation is skipped or the pattern is
  truncated, a notice appears under the title — increase the spacing or disable a layer.
