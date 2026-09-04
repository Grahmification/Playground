# Moving the stock simulation onto the GPU

A plan to replace the CPU stock simulation in `g-code viewer.html` with a GPU one, keeping the
current visual output. Written to be executable by someone who has not seen the earlier work.

---

## 1. Constraints, and why WebGL2 rather than WebGPU

The file is opened directly from disk (`file://`), with no server. That decides the technology:

- **ES modules do not load from `file://`.** A module script is fetched with CORS, and the `file:`
  scheme is not a valid CORS origin, so every `import` fails. Classic `<script src="https://cdn…">`
  tags still work, which is how the file already loads three.js r128.
- **three.js's WebGPU renderer ships only as ES modules.** There is no UMD build of
  `three.webgpu.js`. So WebGPU via three.js is unreachable here.
- Raw WebGPU *is* reachable — `navigator.gpu` is exposed on `file://` because it counts as a secure
  context — but using it means dropping three.js and rewriting the camera, OrbitControls, toolpath
  lines, tool mesh, view cube and grid. Thousands of lines, for no gain on this workload.
- **The gain would be zero anyway.** WebGPU's advantage is compute shaders. The carve is
  "minimum height per grid sample over many swept tool volumes", which maps exactly onto
  rasterisation with a depth test — fixed-function silicon that WebGL2 drives just as directly.

**Use WebGL2, keep three.js r128, share its context.** Everything below needs no extension beyond
core WebGL2, and no float render targets.

Other `file://` rules to respect: no `fetch`, so all shaders are inline template strings; no
workers with module scripts; textures are uploaded from typed arrays, never loaded as images.

---

## 2. What exists today

All stock code is in `g-code viewer.html`. The model is a **2.5D height field sampled at grid
vertices**, `(nx+1) × (ny+1)` samples, with four parallel arrays:

| Array | Meaning |
|---|---|
| `heights` | surface Z at each sample |
| `cutIds` | which operation last lowered that sample (`opIndex+1`; `RAPID_CUT_ID` = gouge) |
| `phi` | signed distance in XY to the governing cut boundary, negative inside, clamped to `±phiBand` |
| `phiOwner` | Z of the wall that owns `phi[i]` — the rim height of the cut that wrote it |

`phi` exists because point sampling loses the sub-cell position of a cut boundary; it is what lets a
wall be drawn somewhere other than a cell edge.

Functions, in the order the pipeline uses them:

| Line | Function | Role |
|---|---|---|
| 2423 | `rebuildStock` | allocates the grid and arrays |
| 2526 | `buildStockGeometry` | static base mesh: `(nx+1)²` verts, 2 tris/cell, 4 skirts, 1 bottom quad |
| 2794 | `buildStockOverlay` | dynamic contour-overlay mesh (vertex pool) |
| 3332 | `carveCapsule` | **the hot loop** — per sample: min height, and the `phi`/`phiOwner` update |
| 3488 | `carveSweep` | slices a Z-changing move into constant-Z capsules |
| 3514 | `carveTo` | walks `parsedSegments`, clips each to the covered arc length |
| 2859 | `scanContourCells` | marks cells the boundary steps through, over the carve's dirty rect |
| 2929 | `rebuildStockOverlay` | rebuilds the overlay: 4-fan marching triangles per contour cell |
| 3144 | `resetStockHeights` | full reset — **runs once per frame while scrubbing backwards** |
| 3199 | `refreshStockGeometry` | uploads all vertex Z, then a full-grid per-cell colour/alpha pass |

Rendering: `stockMaterial` (2275) is a `MeshStandardMaterial` with `flatShading` and `alphaTest`,
coloured entirely by a per-**cell** `DataTexture`; `stockOverlayMaterial` (1306) is the same with
`vertexColors`. Both rely on flat shading deriving normals from screen-space derivatives, so the
normal attribute is a placeholder and is never read.

### Where the time goes

1. `carveTo` → `carveSweep` → `carveCapsule` is O(swept area) in JS, on the main thread. A long
   program takes seconds.
2. **Any backward scrub is a full re-carve from t=0** (`carveTo:3517`), once per frame while the
   slider is dragged.
3. `refreshStockGeometry` rewrites and re-uploads every vertex Z each dirty frame (~17 MB at Ultra)
   and runs a full-grid per-cell colour pass (~1.4 M iterations).
4. Grid resolution is capped at 1600/axis because of 1–3, not because of anything visual.

---

## 3. Target architecture

Every array becomes a GPU texture, produced by rasterising one instanced quad per toolpath move.
Nothing about the stock crosses back to the CPU except a small tile bitmap and a stats reduction.

```
parsedSegments ──(once, at parse)──▶ instance buffer  [ax ay bx by | z toolIdx cutId _]
                                             │
      ┌──────────────────────────────────────┴───────────────────────────────┐
      │  three carve passes, each ONE instanced draw over moves [0, k)        │
      │                                                                       │
      │   A. height   depth24 = min height        + RGBA8 colour = cutId      │
      │   B. owner    depth24 = min rim height                                │
      │   C. phi      depth24 = min signed dist   (discards using B)          │
      └──────────────────────────────────────┬───────────────────────────────┘
                                             ▼
                       D. cell pass  (one full-screen quad over nx × ny)
                          RGBA8 texCell = resolved colour + flags(hasStep, through)
                                             │
                    ┌────────────────────────┼────────────────────────┐
                    ▼                        ▼                        ▼
        E. base mesh (VTF)        F. overlay (tile-instanced)   G. tile mask →
           2 tris/cell, static       20 tris/cell, active          async readback
           discards flagged cells    tiles only                    → active tile list
```

Per frame during playback: **3 instanced draws + 1 full-screen pass + 2 mesh draws.** No vertex
uploads, no colour-texture uploads, no per-sample JS.

Scrubbing to an arbitrary point becomes `instanceCount = k` — the same cost forwards, backwards, or
jumping. The full-re-carve-on-rewind problem disappears entirely.

---

## 4. Key design decisions

### 4.1 The depth buffer does the minimum, and the argmin, for free

Every carve output is a minimum over passes. Rather than float blending (`gl.MIN` needs
`EXT_float_blend` on 32F targets), write the quantity to `gl_FragDepth` and let the depth test do it:

```glsl
gl_FragDepth = clamp((height - uBottom) / (uTop - uBottom), 0.0, 1.0);
```

with `depthFunc(LEQUAL)`, depth writes on, cleared to `1.0`. The depth texture then holds the
minimum height, normalised. Three things fall out:

- **`DEPTH_COMPONENT24` is enough and is core.** 24 bits over a 50 mm block is 3 nm. No float
  render targets, no extensions, no compatibility branches.
- **Clamping is the behaviour we want.** A cut below the block floor clamps to 0, a tool above the
  stock clamps to 1 — exactly `Math.max(target, stockBottom)` in the current code.
- **The colour attachment gets the argmin for free.** The fragment that survives the depth test is
  the one with the lowest height, so writing `cutId` to a colour attachment in the same pass gives
  "the id of the pass that reached deepest", which is what `cutIds` means. `LEQUAL` rather than
  `LESS` makes the *later* operation win ties, matching the current rule at `refreshStockGeometry`.

Read the values back out by sampling the depth texture and rescaling.

### 4.2 `phi` becomes order-independent

The CPU rule is sequential — a running `phiOwner` decides whether each pass replaces or unions.
That cannot be expressed as a min. Reformulate:

```
ownerFinal(s) = min over all passes touching s of rimBase
phi(s)        = min over passes touching s with rimBase <= ownerFinal(s) + levelTol of (d - r)
```

Two passes: B computes `ownerFinal` (a min → depth test), then C computes `phi` (a min → depth
test) with fragments discarded where `rimBase > ownerFinal + levelTol`, sampling B's result.

This is not just a port, it is a **simplification of the semantics**, and it is at least as correct.
Verify the cases before assuming: facing-then-pocket and pocket-then-facing give the same answer;
a ramp is still one region because `ownerFinal` is per sample and only passes whose band covers a
sample can contribute to it; a stepped pocket keeps both walls because the two levels' bands do not
overlap. The `phiOwner` ratcheting in the CPU version was a workaround for the running comparison
and has no equivalent here — **delete it, do not port it.**

`levelTol = WALL_LO * max(cellX, cellY)`, unchanged (see `WALL_LO` at line 1326).

### 4.3 One instanced quad per move

`parsedSegments` is converted once, at parse time, into an interleaved instance buffer. Arcs are
already chorded by the parser, so every entry is a straight segment.

Z-varying moves (ramps, plunges, helical arcs) are **pre-sliced into constant-Z capsules at parse
time**, reusing the existing `carveSweep` slice rule (`zTol = min(cellX, cellY)`, cap 4000). Doing
it once at parse rather than per frame is the whole point; note that the slice count depends on cell
size, so changing stock resolution requires rebuilding the instance buffer — which is already a full
rebuild path.

Instance layout, two `vec4` attributes with `vertexAttribDivisor(1)`:

```
aSeg  = (ax, ay, bx, by)         // segment endpoints in world XY
aInfo = (z, toolIndex, cutId, 0) // tip height, tool row, operation id
```

Tool parameters live in a small `RGBA32F`-free lookup: a `nTools × 1` `RGBA8`-encoded texture is
awkward, so use a **uniform array** `uniform vec4 uTools[64]` = `(radius, flatRadius, cornerRadius,
invTanWall)`. `invTanWall > 0` marks a cone. Programs with more than 64 distinct tools do not exist;
assert and clamp.

The quad is derived from `gl_VertexID` with no vertex buffer at all (raw programs are
`#version 300 es`, so `gl_VertexID` is available). Emit an **oriented** box around the capsule —
along the segment from `A - rOut·dir` to `B + rOut·dir`, half-width `rOut`, plus half a texel of
padding — not an axis-aligned bbox, which is several times larger for a diagonal move.

### 4.4 The grid-to-clip mapping must be exact

The state textures are `(nx+1) × (ny+1)`, one texel per **sample**, and texel centres must land
exactly on sample positions or every wall shifts half a cell. The orthographic bounds are:

```
left   = minX - 0.5 * cellX
right  = left + (nx + 1) * cellX
bottom = minY - 0.5 * cellY
top    = bottom + (ny + 1) * cellY
```

so texel `(i, j)` centre ↔ world `(minX + i·cellX, minY + j·cellY)`. Write this as a helper and use
it for every carve pass. Verify it by carving a single axis-aligned capsule and checking that the
cut spans exactly the sample range the CPU version cuts.

### 4.5 Colour resolves once per cell, not once per sample

The current design deliberately uses **one texel per cell** so that a wall quad reads a single
texel and is one flat colour (see the comment at `buildStockGeometry:2535`). The GPU state textures
are at *sample* resolution, so a per-cell resolve pass is still needed.

Pass D runs one full-screen quad over an `nx × ny` `RGBA8` target and, per cell, reads the four
corner samples and applies the existing rules:

- **RGB** = palette colour of the deepest corner's `cutId`, ties to the larger id
  (`refreshStockGeometry:3255`). Palette lookup uses a `256 × 1` `RGBA8` texture rebuilt from
  `stockColorForCutId` (line 2313) whenever the colour mode changes — 1 KB instead of the current
  full-grid repaint.
- **A** = a bitfield: bit 0 `hasStep` (the `edgeIsStep`/`cellHasStep` test at 2839/2848, ported
  verbatim), bit 1 `through` (highest corner at or below the block floor).

The base mesh discards on either bit; the bottom quad discards on bit 1 only.

### 4.6 The overlay draws only active tiles

The overlay geometry is per contour cell, and contour cells are O(perimeter). Drawing a fixed
template for every cell would cost 60 vertices × 4 M cells, so the active set has to be found.

- Pass G reduces `texCell`'s `hasStep` bit over **8 × 8 tiles** with a max, into an `R8` texture of
  `ceil(nx/8) × ceil(ny/8)`.
- That is read back **asynchronously** — `readPixels` into a `PIXEL_PACK_BUFFER`, then `fenceSync`
  and `clientWaitSync(0)` polled on later frames, `getBufferSubData` when signalled. Never a
  blocking read. 64 KB at a 2048 grid.
- The CPU builds an `Int32Array` of active tile indices and uploads it as a per-instance attribute.
- The overlay draws `drawArraysInstanced(TRIANGLES, 0, 64 * 60, activeTileCount)`.

The readback lags by a frame or two. **Resolve the mismatch by keeping both consumers on the same
generation of the mask**: the base mesh samples the `texCell` from the frame the tile list was built
from, not the newest one. Then a cell is never hidden by the base mesh without the overlay covering
it. Double-buffer `texCell` for this. Additionally dilate the tile list by one tile in each
direction as cheap insurance.

### 4.7 The overlay shader is a direct port of `rebuildStockOverlay`

Do not redesign it. `rebuildStockOverlay` (2929) already implements the exact semantics — 4-fan
marching triangles around a centre vertex, the per-edge crossing rule in `overlayCrossing` (2887),
the wallness blend, the diagonal-pair centre height, through-cut floor skipping, the split colour
rule. Port that logic into the overlay **vertex shader**, with:

- a fixed budget of **20 triangles per cell** (4 sub-triangles × (3 surface + 2 wall)), so a static
  attribute buffer of `64 cells × 60 vertices` holds `(cellInTile, vertexIndex)`;
- unused vertices collapsed to a single point so the triangle is degenerate and culled;
- the same four corner fetches (height from pass A's depth texture, phi from pass C's, id from pass
  A's colour) driving everything.

The properties that make the current version watertight must survive the port, so preserve them
deliberately: **every crossing is a function of the two samples it lies between and nothing else**,
which is why two cells sharing an edge agree to the bit; and **cell membership is judged only on the
four cell edges**, which is why a crossed edge can never have the overlay on one side and the base
grid on the other. If either is broken the seams crack open.

### 4.8 Lighting stays with three.js

Do not reimplement the lighting rig. Patch `stockMaterial` and `stockOverlayMaterial` with
`onBeforeCompile`:

- vertex: replace `#include <begin_vertex>` to read Z from the height depth texture
  (`texture2D` in the vertex shader — vertex texture fetch, universally available) instead of the
  position attribute's Z.
- fragment: replace `#include <map_fragment>` to sample `texCell` by cell coordinate computed from
  the world position, and `discard` on the flag bits.

`onBeforeCompile` shaders are GLSL ES 1.00 in r128, so no `texelFetch` and no `gl_VertexID` there —
sample with `texture2D` at exact texel centres with `NearestFilter`. The **raw programs we write
ourselves** (carve, cell pass, reductions, overlay) are separate GL programs and can be
`#version 300 es`.

This keeps `flatShading` deriving normals from derivatives, keeps the four-light camera-relative rig
in `updateInspectionLights` (1058), and keeps the stock shading identical to today.

---

## 5. GPU resources

Grid `W = nx+1`, `H = ny+1` samples; `nx × ny` cells.

| Resource | Size | Format | Written by | Read by |
|---|---|---|---|---|
| `rtHeight` | W × H | depth24 + RGBA8 colour | pass A | D, E, F |
| `rtOwner` | W × H | depth24, no colour | pass B | C |
| `rtPhi` | W × H | depth24, no colour | pass C | D, F |
| `texCell` ×2 | nx × ny | RGBA8 | pass D | E, F, G, bottom quad |
| `texPalette` | 256 × 1 | RGBA8 | CPU on colour-mode change | D |
| `texTile` | ⌈nx/8⌉ × ⌈ny/8⌉ | R8 | pass G | CPU (async) |
| `texStats` | 64 × 64 → 1 × 1 | RGBA8 | stats reduction | CPU (async, throttled) |
| instance buffer | nMoves × 32 B | — | CPU at parse | A, B, C |

Memory at a 2048 grid: three depth textures ≈ 50 MB, colour ≈ 17 MB, cells ≈ 8 MB. Roughly 75 MB,
against 78 MB of CPU arrays today at 1600.

Use `THREE.WebGLRenderTarget` for A/B/C/D so three owns allocation and disposal, with
`depthTexture: new THREE.DepthTexture(W, H, THREE.UnsignedIntType)` and
`minFilter = magFilter = THREE.NearestFilter`. Bind with `renderer.setRenderTarget(rt)`, then issue
raw GL draws into whatever is bound, then **`renderer.resetState()`** before handing control back —
three caches GL state aggressively and will render garbage otherwise.

Get the context with `renderer.getContext()`. Assert `renderer.capabilities.isWebGL2` at startup; if
it is false, construct the renderer with an explicit context:
`new THREE.WebGLRenderer({ canvas, context: canvas.getContext('webgl2', { antialias: true }) })`.

---

## 6. The carve passes in detail

Shared GLSL, matching `toolHeightAt` (2189) exactly:

```glsl
// t = (radius, flatRadius, cornerRadius, invTanWall); invTanWall > 0 marks a cone.
float toolHeightAt(vec4 t, float d) {
    if (d <= t.y) return 0.0;
    float k = d - t.y;
    if (t.w > 0.0) return k * t.w;
    return t.z - sqrt(max(t.z * t.z - k * k, 0.0));
}

// squared distance from p to segment a-b
float distSqToSeg(vec2 p, vec2 a, vec2 b) {
    vec2 d = b - a;
    float l2 = dot(d, d);
    float t = l2 > 0.0 ? clamp(dot(p - a, d) / l2, 0.0, 1.0) : 0.0;
    vec2 e = p - (a + t * d);
    return dot(e, e);
}
```

**Pass A — height and cut id.** `depthFunc(LEQUAL)`, depth write on, no blending. Cleared to depth
`1.0` and colour `(0,0,0,0)`.

```
d2 = distSqToSeg(worldXY, A, B); if (d2 > r*r) discard;
d  = sqrt(d2);
h  = z + toolHeightAt(tool, d);
gl_FragDepth = clamp((h - bottom) / (top - bottom), 0.0, 1.0);
out = encodeId(cutId);            // 16-bit id across two bytes
```

**Pass B — owner.** Depth only, `LEQUAL`, cleared to `1.0`. `rOut = r + phiBand`.

```
d2 = distSqToSeg(...); if (d2 > rOut*rOut) discard;
rim = max(z + toolHeightAt(tool, r), bottom);
gl_FragDepth = clamp((rim - bottom) / (top - bottom), 0.0, 1.0);
```

**Pass C — phi.** Depth only, `LEQUAL`, cleared to `1.0`. Samples `rtOwner`'s depth texture.

```
d2 = distSqToSeg(...); if (d2 > rOut*rOut) discard;
rim   = max(z + toolHeightAt(tool, r), bottom);
owner = bottom + texture(uOwner, uv).r * (top - bottom);
if (rim > owner + levelTol) discard;
sd = clamp(sqrt(d2) - r, -phiBand, phiBand);
gl_FragDepth = (sd + phiBand) / (2.0 * phiBand);
```

**Draw call.** `gl.drawArraysInstanced(gl.TRIANGLE_STRIP, 0, 4, k)` where `k` is the number of fully
covered moves. The move currently in progress is one extra instance in a separate 32-byte buffer,
rewritten with `bufferSubData` each frame and drawn with `instanceCount = 1`.

**Incremental vs. full replay.** Passes A and B accumulate correctly under the depth test, so during
forward playback they only need the *new* instances. Pass C does not — `ownerFinal` can change, which
invalidates earlier contributions — so it must replay all `k` instances. Start by replaying all three
every time the target distance changes; it is three draw calls and the simplest thing that is
correct. Only split A and B out as incremental if profiling says pass C is not already the floor.

---

## 7. Staged implementation

Each stage leaves the file working and visually checkable. **Keep the CPU carve running in parallel
through stages 1–4** — it costs nothing to leave in place, and it is the reference to diff against.
Do not build any readback plumbing for it; delete it at stage 5 instead.

**Stage 1 — plumbing.** Assert WebGL2, get the raw context, create the render targets, write the
grid-to-clip helper, build the instance buffer at parse time (including the ramp pre-slicing), and
add a `renderer.resetState()` discipline around raw draws. No visual change.
*Checkpoint:* render a debug full-screen quad showing the (still empty) height texture; confirm the
viewport, orientation and Y direction are right by clearing it to a gradient.

**Stage 2 — carve passes A, B, C.** Nothing consumes them yet.
*Checkpoint:* add a temporary debug key that reads pass A back (via a small blit that packs depth
into RGBA8 — 24 bits is exactly three bytes, so this is lossless) and diffs against `stock.heights`.
Every sample must agree to within one depth quantum on the example files. Do the same for `phi`.
This is the correctness gate for the whole plan; do not proceed past a failing diff.

**Stage 3 — cell pass D and the base mesh.** Patch `stockMaterial` with `onBeforeCompile` so vertex
Z comes from `rtHeight` and colour/discard come from `texCell`. Delete the position and colour loops
in `refreshStockGeometry` (3199) and the `DataTexture` in `buildStockGeometry`. Set
`stockMesh.frustumCulled = false` — the CPU no longer knows the heights.
*Checkpoint:* the base surface is pixel-identical to before with the CPU overlay still drawing on
top. Every colour mode still works, and switching mode is now instant.

**Stage 4 — overlay passes G and F.** Tile reduction, async readback, tile-instanced overlay, port
of `rebuildStockOverlay` into the vertex shader. Switch the base mesh onto the double-buffered
`texCell` generation that matches the tile list.
*Checkpoint:* the money shot. Orbit along a pocket wall at a grazing angle looking for cracks
between the overlay and the base grid, and between adjacent overlay cells. A/B against the CPU
overlay by toggling which one is added to the scene.

**Stage 5 — delete the CPU path.** `carveCapsule`, `carveSweep`, `carveTo`'s per-segment loop,
`scanContourCells`, `rebuildStockOverlay`, `overlayCrossing`, `ovVert`, the overlay vertex pool, the
`heights`/`cutIds`/`phi`/`phiOwner`/`contour`/`contourList` arrays, `resetStockHeights`'s array
fills, `stockGeomDirty`/`stockColorDirty`/`stockPhiDirty`/`stockSuppressDirty`, and the dirty-rect
bookkeeping in `stock`. `carveTo` collapses to "work out `k` and the partial instance".
*Checkpoint:* scrub hard in both directions on the longest example file. Backward scrubbing should
now be indistinguishable from forward.

**Stage 6 — stats.** `Material Removed`, the gouge flag and the through-cut flag were CPU
by-products. Replace with a reduction chain over `rtHeight` into a 64 × 64 `RGBA8` (R = mean removed
fraction, G = max through flag, B = max gouge flag), accumulating in full float inside the shader and
quantising only at the final write, then a throttled async readback (~4 Hz) summed on the CPU.
*Checkpoint:* the percentage tracks the old one within a tenth of a percent, and the red-gouge legend
key still appears and disappears at the right moments.

**Stage 7 — raise the ceiling.** Lift the resolution cap in `rebuildStock` (2447) from 1600 and add
a 2048 option to the `#stock-res` select. Note that the limiter is now the base mesh's triangle
count (2048² = 8.4 M triangles/frame), not the carve — so measure before adding more options, and
consider that the honest next step at higher resolutions is a level-of-detail base mesh, not a
bigger grid.

---

## 8. Verification

No test harness exists; this is a single HTML file opened in a browser. `example 4 - wall quality
test.nc` was built for exactly this and exercises facing-then-pocket ordering, a re-cut pocket wall
at two depths, a 30° diagonal slot, a through bore, a ball-nose trough and a 45° chamfer.

| Case | What must hold |
|---|---|
| Stage 2 diff against `stock.heights` and `stock.phi` | agreement to one depth quantum, on all four example files |
| Circular pocket wall | smooth, sub-cell, unchanged from the CPU version |
| Same pocket at every resolution setting | the wall does not move; only facet count changes |
| Ball-nose trough and 45° chamfer | stay smooth — no phantom vertical faces |
| Facing **then** pocket vs pocket **then** facing | identical geometry (this is what §4.2 changes) |
| Ramp and helical entry | slot sides are walls, the ramp face is a smooth slope |
| Through bore (set Stock Bottom to −10 manually; auto-fit takes it from the deepest move) | opens top and bottom, no membrane, no z-fighting |
| Grazing-angle orbit along every wall | no cracks, no background bleed |
| Scrub backwards, forwards, and jump to 0 | identical result to arriving there forwards |
| Colour modes, tool changes, `use-file-tools` toggle | unchanged behaviour, no full-grid repaint |

Measure with `performance.now()` around the carve dispatch and the frame, and check both an
integrated and a discrete GPU if available — the vertex cost of the base mesh is the thing most
likely to differ between them.

---

## 9. Risks

- **State leakage between three.js and raw GL.** By far the most likely source of "everything is
  black". Wrap every raw draw in a save/restore or call `renderer.resetState()` after, and bind
  render targets through `renderer.setRenderTarget` rather than raw `bindFramebuffer`.
- **The half-texel mapping (§4.4).** Getting it wrong shifts every wall by half a cell and will look
  almost right. The stage 2 diff catches it; nothing else will.
- **`gl_FragDepth` disables early-Z.** Expected and unavoidable here, but it means the carve passes
  are fully fragment-bound — keep the instance quad tight (§4.3).
- **`DepthTexture` sampling in a GLSL ES 1.00 `onBeforeCompile` shader.** Works, but confirm
  `checkFramebufferStatus` and that the depth texture is not also bound as the current depth
  attachment when sampled.
- **The order-independent `phi` (§4.2) is a semantic change, not a port.** Walk the cases in that
  section before writing code, and A/B against the CPU version at stage 4 rather than trusting it.
- **Overlay seam integrity (§4.7).** The two invariants there are the whole reason the current
  version has no cracks and no fallback cases. Losing one in translation produces artifacts that
  only show at grazing angles.
