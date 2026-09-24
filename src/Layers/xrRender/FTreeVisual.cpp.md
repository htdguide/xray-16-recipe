# src/Layers/xrRender/FTreeVisual.cpp

> Wind as four shader constants shared by every tree in the frame, lighting as a per-model scale and bias, and a placement transform that keeps the quantized vertices honest.

**Needs** — [`FTreeVisual.h`](FTreeVisual.h.md) · [`xrEngine/Environment.h`](../../xrEngine/Environment.h.md) · [`R_Backend_tree.h`](R_Backend_tree.h.md) · [`xrCore/FMesh.hpp`](../../xrCore/FMesh.hpp.md)
**Used by** — reached through its declarations in [`FTreeVisual.h`](FTreeVisual.h.md); callers name that, not this file.
**Tier floor** — T1: it writes named shader constants per draw and reads a frozen record as a byte image.

## Purpose

Trees are drawn as ordinary meshes whose vertices are displaced by a vertex program. This file is everything that program needs: the wind, recomputed once per frame and shared by every tree; the model's placement, which the quantized vertices are expressed relative to; and the lighting, baked per model into a scale and a bias.

## The tree definition — frozen

```text
RECORD TreeDefinition            # one per model, read as a byte image
  transform    : matrix 4x4      # the model's placement in the world
  scale        : LightingTriple  # multiplies the vertex's stored lighting
  bias         : LightingTriple  # adds to it

RECORD LightingTriple
  rgb  : vector3
  hemi : real
  sun  : real
```

**Invariants**

- **Both the scale and the bias are halved on load**, every component. This converts the authored range — the tool writes values covering a signed unit range — into what the vertex program's packed 8-bit lighting channels expect, which is a half-scale. It is a fixed conversion applied once and a rebuild must apply it or every tree in the game is twice as bright.
- The placement transform is in the *model*, not supplied by the caller. A tree is authored at a fixed world position and its vertices are quantized relative to that position — which is what makes the 16-metre quantization range sufficient. The consequence is that a tree model is not instanceable: two trees are two models with two transforms. That is a real cost and it is the price of the quantization.

## The per-frame wind

**Contract** — computed once per frame, on first use, and reused by every tree drawn that frame. Read from the weather system's current keyframe.

```text
FUNCTION compute_wind_for_this_frame()
  # The wind direction ROTATES continuously, one full turn every
  # `rotation_period` seconds, rather than being a fixed vector.
  angle = 2*pi * global_time / keyframe.tree_rotation_period
  wind  = normalize(sin(angle), 0, cos(angle)) * keyframe.tree_amplitude

  # The wave is three authored spatial frequencies plus a clock in the
  # fourth slot; dividing the whole thing by 2*pi is the vertex program's
  # convention, so the program can use a raw fractional part instead of a
  # trigonometric reduction.
  wave  = (keyframe.tree_wave.x, .y, .z, global_time * keyframe.tree_speed) / (2*pi)

  scale = 1 / vertex_quantum
```

**Invariants**

- **The wind direction rotates rather than gusting.** There is no noise, no gust model and no per-tree phase in the engine; all the variation comes from the three spatial frequencies in the wave, which make trees at different world positions sway out of phase with one another. That is the whole of the tree animation model and it is remarkably cheap.
- The per-frame computation is guarded by a frame number on a single shared record. Every tree in the frame reads the same wind. This is correct — wind is global — and it means the per-tree draw cost is four constant writes and nothing else.
- The wave's fourth component is a *time*, growing without bound as the session runs. Divided by two pi and fed to a fractional-part operation in the vertex program, it eventually loses precision — at 32-bit float, visibly so after a few hours of continuous play. Nothing resets it. This is a real long-session defect and a rebuild should wrap the clock.

## `render(command_list, ...)` — the base

**Contract** — sets the tree constants and draws nothing. The two subclasses call it and then issue their own draw.

```text
FUNCTION render(command_list)
  ensure this frame's wind is computed
  push the model's transform
  push the view-space form of that transform          # newer renderers only
  push (vertex scale, vertex scale, 0, 0)
  push the wave and the wind
  push the lighting scale and bias, multiplied by a console-tunable brightness
  push (scale.sun, bias.sun, 0, 0)
```

**Invariants**

- The brightness multiplier is one console value on the oldest renderer and **the same value times 4/3** on the newer ones. The ratio compensates for the difference between the forward path's lighting and the deferred path's; it is a hand-matched constant with no derivation, and it exists so that one authored tree looks the same under both renderers.
- The oldest renderer additionally **adds the current weather ambient into the lighting bias**, because it has no separate ambient term in its material. The newer ones apply ambient later, in the lighting pass. Same authored data, two places it is consumed.
- The view-space transform is pushed only on the newer renderers, which need it to write view-space normals into the g-buffer. It is composed here rather than in the program because it is per model and the program is per vertex.

## `TreeVisual_Static` and `TreeVisual_Progressive`

**Contract** — the still variant draws the whole mesh in one indexed draw. The simplifying variant chooses a slide window exactly as the progressive static model does — inverted level-of-detail value, round to nearest, remember the last one for a negative argument — and draws that range. Both account their vertices to the **flora** bucket of the draw statistics, which is why the developer overlay reports trees separately from everything else.

**Invariants** — The simplifying variant's slide-window table is **shared through the level**, fetched by id, not owned and not copied. Unlike [`FProgressive.cpp`](FProgressive.cpp.md), this is correct: many trees of one species in a level share one table, and the level owns it. The inconsistency between the two types is historical.

## The named constants

**Notes** — The eight constant names are interned into file-scope variables the first time any tree loads, and then re-interned on every subsequent load — the assignment is unconditional. It is harmless (interning an existing string is a lookup) and pointless. What survives is that the tree vertex program's interface is a fixed set of eight named constants, and that naming them by string rather than by slot is how the material system binds constants at all; see [`r_constants.cpp`](r_constants.cpp.md).

There is a disabled block in the load path that would have baked the placement transform into the vertices for the fixed-function path, which cannot apply a vertex program. It is incomplete — the loop body does nothing but advance — and the fixed-function path consequently draws trees at the model origin. This is a known-broken configuration, not a recoverable one.
