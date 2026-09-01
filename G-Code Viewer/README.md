# G-Code Viewer

A 3D viewer for CNC milling programs. It draws the toolpath, plays it back like a
video, and carves a block of stock as it goes so you can see the shape the program
cuts.

It is one HTML file. There is nothing to install and no server to run.

## Running it

Open `g-code viewer.html` in a browser. That is all.

It needs an internet connection the first time, because it loads three.js from a CDN.

Some example programs are in this folder if you want something to try:

- `example 1 - helical bore.nc`
- `example 2 - multiple tools.nc`
- `example 3 - 3D contouring.nc`

## Using it

**Open a file** with the button at the top of the sidebar. Accepted extensions are
`.gcode`, `.nc`, `.cnc`, `.tap` and `.txt`.

**Moving the view:**

| Action | Mouse |
| --- | --- |
| Rotate | Left drag |
| Pan | Right drag |
| Zoom | Scroll wheel |

The cube in the bottom right corner snaps to a standard view. Click a face, an edge
or a corner.

**Playback** is the bar at the bottom of the 3D view:

| Button | What it does |
| --- | --- |
| Reset | Back to the start |
| Back / Forward | Jump 1 second of machining time |
| Play / Pause | Run or stop the simulation |
| Speed | 0.01x to 50x. 1x is real time |

The slider under the 3D view scrubs to any point in the program. Its coloured bands
are the operations, sized by how long each one takes.

**The panel in the top right** shows the active tool, the current feed, the spindle
speed and where the tool is. Under it is the line of the program being run. Click the
arrow to see the lines coming up next.

### Sidebar

**Toolpath Lines** — colour the lines by motion type, feed rate, tool or operation.
You can hide the lines, or show only the bit being cut right now.

**Stock Simulation** — tick the box to show a block of material and cut it away as
the program runs. You need a tool size for it to remove anything: either the file
lists its tools (see below) or you type one in under Manual Tool.

- *Stock Resolution* is how fine the block is sampled. Higher looks better and runs
  slower.
- *Stock Margin* is how far the block sticks out past the toolpath. Negative pulls it
  inside.
- *Stock Top Z* and *Stock Bottom Z* set how thick it is.
- *Reset Stock To Fit Job* puts all of these back to a guess based on the program.

Red on the stock means a rapid (G0) cut material. That is almost always a mistake in
the program.

**Manual Tool** — the cutter to use when the file does not name one. Flat, ball, bull
nose and chamfer/V-bit.

**File Information** — file name, command count, feed range and the size of the job.

On a narrow screen, the arrow at the top of the sidebar folds it away.

## How your file should be written

Plain text, one command per line. Normal CAM output works. Casing does not matter.

Comments are `( round brackets )` or anything after a `;`. Both are ignored.

Everything is treated as **millimetres**. `G20` (inch mode) is not handled, so an inch
program will draw the right shape but every number on screen will be wrong.

Positions are absolute by default (`G90`). `G91` relative works too.

Arcs must use **I and J** to give the centre. R-word arcs are not supported. Arcs are
always read as being in the XY plane, so `G18` and `G19` arcs will come out wrong.

### Supported commands

Anything not in this list is ignored. Unknown commands do not break the file, they
just do nothing.

**Motion**

| Code | Meaning |
| --- | --- |
| G0 | Rapid |
| G1 | Feed |
| G2 | Clockwise arc (needs I / J) |
| G3 | Counter-clockwise arc (needs I / J) |

**Drilling cycles**

| Code | Meaning |
| --- | --- |
| G73 | Peck drill, short chip break |
| G81 | Drill |
| G82 | Drill with dwell (drawn the same as G81) |
| G83 | Peck drill, full retract |
| G80 | Cancel the cycle |
| G98 | Retract to the height the cycle started at |
| G99 | Retract to the R plane |

Cycles use `R` for the retract height and `Q` for the peck depth. Each hole is drawn
as the moves a machine really makes: over the hole, down to R, feed to depth, retract.

**Other G-codes**

| Code | Meaning |
| --- | --- |
| G90 | Absolute positions |
| G91 | Relative positions |

**M-codes**

| Code | Meaning |
| --- | --- |
| M3 / M4 | Spindle on |
| M5 | Spindle off |
| M2 / M30 | End of program, spindle off |
| M6 | Tool change |

**Words**

| Word | Meaning |
| --- | --- |
| X Y Z | Position |
| I J | Arc centre, relative to the start of the arc |
| F | Feed rate |
| S | Spindle speed |
| T | Tool number |
| R | Retract height, in a drilling cycle |
| Q | Peck depth, in a drilling cycle |

If your file uses `M6`, a tool only changes on the `M6` line. If it never uses `M6`,
a bare `T` word changes the tool on its own. Same idea for the spindle: if the file
never uses `M3`/`M4`/`M5`, the spindle is treated as running the whole time.

### Optional: tool sizes in the header

If the file lists its tools in a comment block at the top, the viewer will use the
real cutter shapes and switch between them at each tool change. This is what most CAM
programs already write. The format is:

```
(T4  D=12.7 CR=0. - ZMIN=-13.5 - flat end mill)
(T11  D=4. CR=0. TAPER=90deg - ZMIN=-1.5 - spot drill)
(T12  D=6.35 CR=0. TAPER=45deg - ZMIN=-14.6 - chamfer mill)
(T19  D=5. CR=0. TAPER=118deg - ZMIN=-17.502 - drill)
```

- `D=` diameter, `CR=` corner radius, `TAPER=` point angle. `D=` must be there.
- The text at the end sets the shape. Words it looks for: *spot*, *countersink*,
  *chamfer* / *taper*, *ball*, *bull* / *radius*, *drill*, *tap*, *thread*, *ream*,
  *bore*, *probe*, *mill*.

Untick *Use tool geometry from G-code file* to ignore the header and use the Manual
Tool instead.

### Optional: operation names

A comment on an `N` line names the operation. It shows in the legend and on the
timeline:

```
N10(Facing - 0.5 EM)
```

Without these, each tool change starts a new operation.

## Things to know

- Only 3-axis milling. No lathe, no rotary axes, no tool compensation (`G41`/`G42`).
- Work offsets (`G54` and friends) are ignored. Everything is drawn in one
  coordinate system.
- The stock is a height map, so it can only be cut from above. Undercuts cannot be
  shown.
- G-code never says where the tool starts, so the viewer begins directly above the
  first point the program moves to.
