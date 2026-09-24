# src/Layers/xrRender/ModelPool.cpp

> The renderer's model cache: one loaded copy of each model file, cheap clones of it for each object that wears it, and a free list so a clone that dies is reused instead of reloaded.

**Needs** — [`ModelPool.h`](ModelPool.h.md) · [`FBasicVisual.h`](FBasicVisual.h.md) · [`FVisual.h`](FVisual.h.md) · [`FProgressive.h`](FProgressive.h.md) · [`FSkinned.h`](FSkinned.h.md) · [`FHierrarhyVisual.h`](FHierrarhyVisual.h.md) · [`FLOD.h`](FLOD.h.md) · [`FTreeVisual.h`](FTreeVisual.h.md) · [`SkeletonCustom.h`](SkeletonCustom.h.md) · [`SkeletonAnimated.h`](SkeletonAnimated.h.md) · [`SkeletonX.h`](SkeletonX.h.md) · [`ParticleEffect.h`](ParticleEffect.h.md) · [`ParticleGroup.h`](ParticleGroup.h.md) · [`xrCore/FMesh.hpp`](../../xrCore/FMesh.hpp.md) · [`xrMaterialSystem/GameMtlLib.h`](../../xrMaterialSystem/GameMtlLib.h.md) · [`xrEngine/IGame_Persistent.h`](../../xrEngine/IGame_Persistent.h.md)
**Used by** — reached through its declarations in [`ModelPool.h`](ModelPool.h.md); callers name that, not this file.
**Tier floor** — T1: it reads the first chunk of a shipped model file as a byte image to learn the model's type before it knows which record to build, and the visuals it hands out own device buffers whose release order matters.

## Purpose

Hundreds of objects in a level wear the same model. Loading the file once and sharing it is not enough, because each wearer needs its own animation state, its own bone transforms and its own bounding box — but the *geometry*, the material assignments and the bone hierarchy are identical and expensive. This file owns the three-level scheme that resolves that: a **base** model per file, a **clone** per wearer that shares the base's heavy data and owns its light data, and a **pool** of clones whose wearer has died but which are likely to be wanted again in a moment.

It is also the one place that knows the mapping from a model file's recorded type to the concrete visual record that implements it. Everything else in the renderer receives a visual and asks it questions.

## State

```text
RECORD ModelDef                   # one loaded model file
  name  : text                    # normalized: lowercased, extension stripped
  model : Visual                  # the base; never rendered, only copied
  refs  : int                     # how many clones (live or pooled) descend from it

RECORD ModelPool
  models      : list<ModelDef>            # the bases, searched linearly by name
  registry    : map<Visual, text>         # every clone -> the base name it came from
  pool        : multimap<text, Visual>    # clones whose wearer released them, by base name
  to_delete   : list<Visual>              # deletions deferred because a frame is being drawn
  logging            : bool               # report every uncached load
  force_discard      : bool               # re-entrancy latch, see Discard
  allow_child_duplicate : bool            # re-entrancy latch, see CreateChild
```

**Invariants**

- Every visual in `pool` is also in `registry`; the pool is a subset of the clones, not a separate population. A clone is in exactly one of the two states "held by a wearer" or "in the pool", and `registry` does not distinguish them — pool membership is the distinction.
- A base's `refs` counts clones that exist, including pooled ones. A base is destroyed only when that count reaches zero, so a pooled clone keeps its base alive. This is deliberate: the pool exists precisely because the model is expected to be wanted again.
- A visual that is *not* in `registry` was never cloned from a base — a particle effect compiled from a definition, a hand-built visual — and is simply destroyed when released rather than pooled.
- No model may be created or destroyed while a frame is being drawn. Creation is not guarded (callers are expected not to); destruction is, by deferral.
- Names are the cache key and are normalized on every entry point: lowercased, any extension removed. Two spellings of the same file must collide.

## `Create`

**Contract** — given a model name (and optionally an already-open reader for its bytes), returns a visual the caller owns until it passes it back to `Delete`. Never returns nothing: a missing file is fatal in the game (the authoring tools downgrade it to a message and a null). Blocks on file I/O when the model is not already loaded. This is the entry point the game uses for every object with a visual.

```text
FUNCTION create(name, optional data) -> Visual
  key = normalize(name)                       # lowercase, drop extension

  IF pool has an entry under key
    take one out, tell it to Spawn            # reset per-wearer state; geometry is already there
    RETURN it

  base = find base named key                  # linear scan of the bases
  IF base is none
    allow_child_duplicate = false             # see below
    base = load(key, data, register = true)
    allow_child_duplicate = true

  clone = duplicate(base)                     # copy, then Spawn
  registry[clone] = key
  RETURN clone
```

**Notes**

- The pool is consulted *before* the base list, because a pooled clone is already in the state a wearer wants and reusing it skips the copy entirely. This is the whole point of the pool: in a firefight, corpses and bullet-hole decals are created and destroyed at a rate where the copy cost is visible.
- `Spawn` is the clone's own reset hook — it is what distinguishes "a fresh instance" from "the bytes of one". A rebuild must keep the two apart: copying is about data, spawning is about state.

## `CreateChild`

**Contract** — same as `Create`, but for a visual that is a *component* of another visual (a level-of-detail mesh, a skinned sub-mesh, a hierarchy member) rather than something an object wears. Does not enter the registry and is not pooled; its lifetime belongs to its parent.

**Notes** — The latch: while a base is being loaded, child duplication is switched off, so children resolve to the shared base itself rather than to copies. The reasoning is that a *base* is never rendered and never animated, so its children need not be private; only when a clone is made do its children get copied along with it. Switching a flag on the pool rather than passing a parameter down is an artifact of the load path crossing through the model file reader, which does not carry renderer arguments; a rebuild threads the flag through the load context instead.

## `Instance_Load`

**Contract** — reads a model file, builds the right visual record for the type recorded in its header, and optionally registers it as a base. Resolves the search path itself. Fatal if the file cannot be found. Blocks.

```text
FUNCTION load(name, allow_register) -> Visual
  file = name with ".ogf" appended when it has no extension
  path = first that exists of: name as given, $level$/file, $game_meshes$/file
  FAIL WITH fatal IF none exists

  reader = open(path)
  header = read the model header chunk          # fixed-size byte image, gives the type
  visual = create empty visual of that type
  visual.load(name, reader)
  close(reader)

  IF type is a skeleton (rigid or animated)
    resolve each bone's game material           # see below
  IF allow_register
    append to bases as (name, visual, refs = 0)
  RETURN visual
```

**Notes**

- Two search roots and a default extension: models live either with the level (level-specific geometry) or in the shared mesh archive, and the shipped data references them both ways, sometimes with the extension and sometimes without. A caller-supplied literal path wins over both. All three spellings must resolve, because the shipped configuration files use all three.
- The header is read as a fixed-size record before anything else is parsed, because the file's *type* determines which record the rest of it is. This is the one genuinely T1 moment in the file.
- A base is registered with a reference count of **zero**, not one. The base is not a clone and does not count as a user of itself; if the first clone is destroyed the base goes with it.

### Bone material resolution

Every bone of a skeleton carries an authored surface-material *name*; the physics and hit systems want an *index* into the material table. Resolution happens once, at load, for the whole skeleton:

```text
FOR EACH bone IN skeleton
  IF bone has an authored material name
    bone.material = index of that name
    ASSERT the material is marked dynamic
  ELSE
    bone.material = index of "default_object"
```

`default_object` is a named convention in the shipped material table and must exist and must be marked dynamic — a bone belongs to a moving object by definition, and the pairwise interaction table is keyed on the dynamic/static distinction. The engine asserts rather than substituting, because a static material on a bone produces wrong footstep sounds and wrong ricochet behaviour rather than a visible failure.

## `Delete` and `DeleteQueue`

**Contract** — hands a visual back. The caller's reference is cleared. Either returns the visual to the pool or destroys it, depending on the discard flag and on whether the visual has a base. Safe to call during a frame: the work is deferred to `DeleteQueue`, which the frame loop runs between frames. Discarding *during* a frame is not permitted even deferred — a discard can destroy a base, and a base's device buffers may be bound.

```text
FUNCTION delete(visual, discard)
  IF a frame is being drawn
    ASSERT not discard
    append to to_delete                     # DeleteQueue drains this between frames
  ELSE
    delete_now(visual, discard)

FUNCTION delete_now(visual, discard)
  visual.Depart()                           # the mirror of Spawn: release per-wearer state
  IF discard OR force_discard
    discard(visual, complete = true)
  ELSE IF visual is in registry
    move it into the pool under its base name
  ELSE
    destroy it                              # never had a base: nothing to pool it against
```

**Notes** — `Depart` is called on both paths, including the one that destroys. The pool holds *departed* clones: a pooled clone is inert, and the `Spawn` on the way out is what makes it live again. Keeping the pair symmetric is what makes pooling safe to bolt onto a type that was not designed for it.

## `Discard`

**Contract** — destroys a clone for real and drops one reference from its base, destroying the base too when the last reference goes. Idempotent only in the sense that a visual with no registry entry is simply destroyed.

```text
FUNCTION discard(visual, complete)
  IF visual not in registry
    destroy it; RETURN

  name = registry[visual]
  base = base named name
  IF complete OR name contains "#"
    base.refs = base.refs - 1
    IF base.refs == 0
      force_discard = true                  # every clone released during teardown must also discard
      base.model.Release(); destroy base.model; remove base
      force_discard = false
  ELSE
    base.refs = max(0, base.refs - 1)        # keep the base loaded even at zero users
  destroy visual
  remove from registry
```

**Notes**

- **The `#` convention is load-bearing.** A name containing a hash is a *generated* model — a variant assembled at runtime rather than a file on disk (a weapon with an attachment, a body with a substituted head). Those names are unbounded in number, so caching their bases forever would leak; they are therefore always fully discarded, never kept warm. A rebuild that changes the naming convention must keep some equivalent marker, or the base list grows without limit over a long session.
- The re-entrancy latch: destroying a base tears down its children, and each child comes back through the delete path. Without the latch those children would be *pooled* — added to the pool of a base that is in the middle of being destroyed. The latch forces the whole cascade to discard. It is a symptom of destruction being recursive through the same entry point, and a rebuild that tears down with an explicit worklist does not need it.

## `ClearPool`

**Contract** — discards every pooled clone. With `complete` set, also drops their bases' references so the bases themselves can go; without it, bases survive with their counts decremented. Called at level transitions: the pool's whole value is locality in time, and across a level boundary there is none.

## `Instance_Create`

**Contract** — builds the empty visual record for a model type identifier. Fatal on an unknown type. The mapping is a frozen part of the model format: the type field in the file header selects the record, and the shipped data uses every one of them.

```text
model type            -> visual record
  normal              -> static mesh
  hierarchy           -> a container of child visuals, no geometry of its own
  progressive         -> static mesh with a collapse sequence for dynamic resolution
  skeleton rigid      -> bone hierarchy, no animation player
  skeleton animated   -> bone hierarchy with the animation player on top
  skeleton geometry, static      -> one skinned sub-mesh
  skeleton geometry, progressive -> one skinned sub-mesh that also collapses
  particle effect     -> a single emitter's visual
  particle group      -> a timeline of effects
  level of detail     -> the impostor form of a distant model
  tree, static        -> wind-animated mesh
  tree, progressive   -> wind-animated mesh that also collapses
```

**Notes** — The last three exist only in the game build; the authoring tools never encounter them. That is a build-configuration fact, not a design one — a rebuild should support all of them everywhere.

## `Instance_Duplicate`

**Contract** — makes a clone of a base: allocate the same record type, copy, spawn, and charge one reference to the base. The copy is shallow where the data is immutable (geometry buffers, index buffers, the bone hierarchy's static description, materials) and deep where it is per-wearer (bone transforms, animation state, bounding volume). Which is which is decided by each visual type's own copy, not here.

## `CreatePE` and `CreatePG`

**Contract** — build a particle-effect or particle-group visual directly from an in-memory definition rather than from a file, and compile it. These visuals are outside the cache entirely: no name, no base, no pool entry. A particle system's definition is already shared through the particle library; the visual is a thin per-emission instance.

## `Prefetch`

**Contract** — creates and immediately releases every model named in the configuration section for the current game type, so that the pool and base list are warm before play begins. Logging is suppressed for the duration, because every one of these is by definition an uncached load. Blocks for as long as the list is long; this is a loading-screen operation.

**Notes** — The section name is composed as a fixed prefix plus the game-type string, which means the shipped configuration carries one prefetch list per multiplayer mode and single-player. The list is pure policy — it changes load-time distribution, never behaviour — but the section-name convention is frozen against the shipped files.

## `Instance_Register` · `Instance_Find`

**Contract** — append a base under a name; find a base by exact name. The search is linear over all loaded models, which is a few hundred entries and runs only on a cache miss. That it is not a map is a deliberate non-decision; a rebuild should use one and lose nothing.

## `dump` · `memory_stats`

**Contract** — diagnostics. `dump` writes a per-base and per-clone memory report distinguishing pooled ("free") from held ("used") clones; `memory_stats` totals the vertex and index buffer bytes of every base, split into device-resident and system-resident. Both walk everything and are console-triggered, never per-frame.

**Notes** — The split between device and system memory for each buffer is the thing worth preserving: the renderer keeps a system-side copy of some index data for collision and progressive-mesh work, and the report exists so that a level that runs out of video memory can be told apart from one that merely uses a lot.

## Editor-only rendering

**Contract** — in the authoring-tool build the pool also draws a visual directly: walk its children, skip those whose material does not match the requested priority bucket and back-to-front flag, bind each child's material (falling back to a wireframe material when it has none), set the world transform and draw. `RenderSingle` runs the four priority buckets, each twice, once for each setting of the back-to-front flag — eight passes that reproduce, in the dumbest possible way, the ordering that the game's draw stream achieves with a sort key.

**Notes** — This is the clearest statement in the chapter of what the sort key *means*: priority 0..3 outer, back-to-front inner. A rebuild does not need this code, but the ordering it spells out is the one the shipped materials assume.
