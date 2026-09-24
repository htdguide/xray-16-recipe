# src/Layers/xrRender/r__dsgraph_types.h

> The bucket types the frame's visible geometry is sorted into: what a queued draw remembers, and how queues are keyed.

**Needs** — [`r__dsgraph_structure.h`](r__dsgraph_structure.h.md) · [`Shader.h`](Shader.h.md) · [`FBasicVisual.h`](FBasicVisual.h.md) · [`xrCore/Containers/FixedMap.h`](../../xrCore/Containers/FixedMap.h.md)
**Used by** — [`r__dsgraph_build.cpp`](r__dsgraph_build.cpp.md) · [`r__dsgraph_render.cpp`](r__dsgraph_render.cpp.md) · [`r__dsgraph_structure.h`](r__dsgraph_structure.h.md)
**Tier floor** — T1: these records are produced by the thousand per frame and sorted repeatedly; their size and the fact that the matrix is stored by value rather than referenced are the point.

## Purpose

Declares the vocabulary of the scene graph. The graph is not a tree — it is a set of **buckets**, and the whole design is in which buckets exist and what each is keyed by. The algorithms that fill them are in [`r__dsgraph_build.cpp`](r__dsgraph_build.cpp.md); the ones that drain them are in [`r__dsgraph_render.cpp`](r__dsgraph_render.cpp.md).

## The queued draw

```text
RECORD StaticItem
  coverage : real       # screen-space area estimate
  visual   : Visual

RECORD DynamicItem
  coverage : real
  owner    : optional<Renderable>   # the object, for per-object shader constants
  visual   : Visual
  transform: matrix4                # BY VALUE

RECORD SortedItem            # a DynamicItem plus the element to draw it with
  coverage : real
  owner    : optional<Renderable>
  visual   : Visual
  transform: matrix4
  element  : ShaderElement

RECORD LodItem
  coverage : real
  visual   : Visual
```

Invariants and the decisions behind them:

- **The transform is stored by value.** A queued draw outlives the walk that produced it — the object may be animated, moved or destroyed between collection and drawing — so the graph may not hold a pointer into the object's transform. Copying a full matrix per queued dynamic visual is the price, and it is the reason the dynamic buckets are heavier than the static ones.
- **Static items carry no transform at all.** Static geometry is authored in world space; its transform is always identity. The two item shapes exist precisely so the static path does not pay the matrix copy.
- `coverage` is the screen-space-area estimate — a visual's bounding radius over its squared distance to the camera. Every sort in the renderer uses it, and every level-of-detail decision thresholds against it, so it must be computed identically everywhere; see [`r__dsgraph_build.cpp`](r__dsgraph_build.cpp.md).
- The "sorted" variant carries its own shader element rather than taking the visual's. The transparency, emissive, distortion and decal buckets each draw the *same* visual with a *different* element, so the element cannot be looked up at draw time.
- The sorted-item record deliberately repeats the dynamic item's fields instead of embedding it, so it can be built from a flat initializer list. That is a language convenience; a rebuild should compose them.

## The buckets

```text
# Opaque geometry, keyed twice: by pass index, then by pass identity.
StaticBucket  = list<StaticItem>  + a running maximum coverage
DynamicBucket = list<DynamicItem> + a running maximum coverage

StaticPasses  = per shader-pass-slot, map<Pass, StaticBucket>
DynamicPasses = per shader-pass-slot, map<Pass, DynamicBucket>

# Everything that must be drawn in depth order, keyed by squared distance.
SortedBucket = ordered map<real, SortedItem>
LodBucket    = ordered map<real, LodItem>
```

The shape decisions:

- **Opaque geometry is keyed by the material pass, not by distance.** Changing a pass — its shaders, its textures, its state — is the expensive operation; the order objects are drawn in within a pass is nearly free. So opaque draws are grouped by pass and only then sorted inside the group.
- **Each bucket remembers the largest coverage it holds.** That is how the *passes themselves* are ordered: a pass containing something big and near is drawn before one containing only distant things, so the depth buffer rejects the most pixels earliest.
- **Passes are indexed by slot first.** A material's element has several passes and they must be drawn in the order the material declares, across all materials — pass zero of everything, then pass one of everything. The outer array is that slot; the inner map is which pass occupies it.
- **The ordered buckets are keyed by squared distance, not distance.** The key is only ever compared, and the square root would be pure cost. Several items may share a key, so the map admits duplicates and preserves insertion order among them.

The ordered maps are walked left-to-right for front-to-back and right-to-left for back-to-front, which is the only reason they are ordered containers rather than lists plus a sort.
