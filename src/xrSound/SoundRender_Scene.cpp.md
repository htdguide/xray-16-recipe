# src/xrSound/SoundRender_Scene.cpp

> One sound world: its emitters, the two geometry databases that answer occlusion and reverb-region
> queries, the play entry points, and the queue that feeds the AI's hearing sense.

**Needs** — [`SoundRender_Scene.h`](SoundRender_Scene.h.md) · [`SoundRender_Core.h`](SoundRender_Core.h.md) · [`SoundRender_Emitter.h`](SoundRender_Emitter.h.md) · [`SoundRender_Environment.h`](SoundRender_Environment.h.md) · [`Common/LevelStructure.hpp`](../Common/LevelStructure.hpp.md) · [`xrCDB/Intersect.hpp`](../xrCDB/Intersect.hpp.md) · [Seam: Static collision database](../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: it reads two level files as memory images, rewriting triangle payload words in
place and reinterpreting a stored word's bits as a real.

## Purpose

A scene is a self-contained sound world. It owns the emitters, the level geometry that makes sound
quieter, the level geometry that says which room you are in, and the queue by which a sound becomes
something a creature can hear. Splitting this out of the manager is what lets the level editor open
several worlds at once and what lets a level's geometry be swapped without touching the voice pool.

Three databases arrive from the level's data, and they are different things:

| database | source | answers |
|---|---|---|
| collision model | the level's own collision mesh, shared with physics | "is there anything at all between listener and source?" |
| **SOM** (sound occlusion mesh) | a dedicated level file | "how much does what is between them attenuate?" |
| **environment mesh** | a dedicated level file | "which reverb preset is this point in?" |

The first two are queried per audible emitter per update; the third only when the listener or a
source moves.

## State

```text
RECORD Scene
  emitters      : list<Emitter>
  hearing_queue : list<(SoundHandle, real)>   # (what was heard, how far it carries)
  prev_event_count : int                      # for the statistics overlay only
  hearing_handler  : optional<callback>       # the AI's ear; set by the game
  collider      : Collider                    # scratch state for ray queries
  geom_collision: optional<Model>             # not owned: the level's collision model
  geom_som      : optional<Model>             # owned, built here
  geom_env      : optional<Model>             # owned, built here
  user_env      : optional<Environment>       # a script override of the whole region system
  pause_depth   : int                         # starts at 1; see below
```

Invariants:

- Every emitter in `emitters` is alive and has this scene as its owner. Emitters are created only
  by `play` and destroyed only by the processor's reap.
- `hearing_queue` holds handles, not emitters, and is drained every update. An emitter that dies
  mid-frame withdraws its own entries first.
- `pause_depth` starts at **1**, not 0, so that the very first "unpause" the game issues at level
  start matches an implicit pause taken at construction. The counter is the *identity* an emitter
  records, not a reference count — see
  [`SoundRender_Emitter_StartStop.cpp`](SoundRender_Emitter_StartStop.cpp.md).

## `play` / `play_at_pos`

**Contract** — Start a sound on a handle, associating it with a game object. If the handle already
has an emitter, *rewind* it instead of creating a second one — one handle is one voice. A stereo
asset, or a caller-requested 2D flag, forces the emitter to 2D. `play_at_pos` additionally places
it. Both are no-ops when no device exists or the handle is empty.

**Invariants** — On return the handle has exactly one emitter and that emitter names the handle.

## `play_no_feedback`

**Contract** — Fire-and-forget: plays the sound on a *private copy* of the handle, so the caller's
handle keeps no link to the emitter and cannot later move or stop it. Optional position, volume,
frequency and range overrides are applied at start. The caller's handle is restored before return.

**Notes** — This is how the game plays one-shots it will never touch again — an impact, a shell
casing — without the bookkeeping of owning them. The copy shares the underlying asset; only the
per-instance record is duplicated. A rebuild with proper value semantics gets this for free and
should not need a separate entry point, but the *behaviour* (the caller cannot reach the sound
afterwards) is load-bearing: it is what makes the sound safe to outlive its owner.

## `i_play`

**Contract** — Creates the emitter, links it both ways to the handle, and starts it. Asserts the
handle has no emitter already. The only place emitters are born.

## `set_geometry_som`

**Contract** — Loads the sound occlusion mesh and builds a query tree over it. Replaces any
previous one. Refuses a file whose version is not the expected one.

```text
# The file is a chunked container: chunk 0 is a version word, chunk 1 is an
# array of records read as a memory image.
RECORD SomPolygon
  v1, v2, v3 : vector
  two_sided  : int
  occlusion  : real      # 0..1 multiplier applied to a ray that crosses this face

FUNCTION set_geometry_som(reader)
  require version = 0
  FOR EACH poly IN chunk 1
    add triangle (v1,v2,v3) carrying occlusion as its payload word
    IF poly.two_sided THEN add the reversed winding too, same payload
  build the query tree
```

**Notes** — The occlusion value is stored in the triangle's payload word *as the bit pattern of a
real*, and read back the same way. The collision database's payload is an opaque word by design,
so this is the intended use — but a rebuild must reproduce the bit-exact round trip, not convert
through an integer.

Two-sided faces are duplicated with reversed winding because the query culls back faces: a single
sheet of geometry representing a curtain must attenuate from both sides, and duplicating it is
cheaper than disabling culling for the whole query.

## `set_geometry_env`

**Contract** — Loads the environment mesh, resolves its per-face preset names against the loaded
preset library, and builds a query tree. Each triangle names a preset for its front side and
another for its back — the two sides of a doorway are two different rooms.

```text
FUNCTION set_geometry_env(reader)
  IF no preset library is loaded THEN RETURN      # no presets, no regions
  names ← chunk 0 as a list of preset names
  ids   ← FOR EACH name: library.id_of(name)      # must resolve; a miss is fatal
  copy chunk 1 into private storage               # it is rewritten in place
  read the collision-format header, then the vertex and triangle arrays
  FOR EACH triangle
    front ← low half of the payload word ; back ← high half
    # Rewrite the file's local indices into library ids, in place.
    payload ← (ids[back] << 16) OR ids[front]
  build the query tree
```

**Notes** — The rewrite is why the chunk is copied first: the file's payload words are *indices
into this file's own name table*, and the query wants *library ids*, so the geometry is translated
once at load rather than indirecting on every query. Two presets are packed into one 32-bit word as
two 16-bit halves, which caps a level at 65 536 presets — far beyond any real level, and the packing
is frozen by the shipped level files.

## `environment_at` — which room is this point in?

**Contract** — Returns the reverb preset governing a world point. A script override wins over
geometry; with no geometry and no override, the identity (reverb-free) preset is returned. One ray
cast. Never returns nothing.

```text
FUNCTION environment_at(point) -> Environment
  IF a script override is set THEN RETURN it
  IF no environment geometry THEN RETURN identity

  # Cast straight down: rooms are defined by the floor you are standing over.
  hit ← nearest ray hit against geom_env from point, direction (0,-1,0), length 1000
  IF no hit THEN RETURN identity

  normal ← the hit triangle's normal
  # Below the face or above it — that is which of the triangle's two presets applies.
  IF dot((0,-1,0), normal) < 0 THEN RETURN library[low half of payload]
  ELSE                              RETURN library[high half of payload]
```

**Notes** — A downward ray, not a containment test. The environment mesh is authored as *floors*,
one per acoustic region, so "which region am I in" reduces to "what am I standing over", which is
one ray instead of a point-in-volume query against overlapping convex hulls. The consequence a
rebuild must accept: a point with no floor beneath it within a kilometre gets the identity preset,
and a region is unbounded upward.

The front/back split means one sheet of geometry serves both sides of a floor — the cellar below and
the room above — without authoring two meshes.

## `occlusion_at` — how much is in the way?

**Contract** — Returns a multiplier in [0,1] for a source at a given point, from the listener's
position. Casts up to two rays. The caller passes a dispersion radius; the ray is aimed at a random
point on a sphere of that radius around the source rather than at the source exactly. Called once
per audible 3D emitter per update, which is the affordability budget for the whole system: at most
one or two rays × the number of emitters that passed the distance test, against an immutable tree
that supports concurrent queries.

```text
FUNCTION occlusion_at(point, radius, cached_triangle) -> real
  factor ← 1
  aim ← point + random_direction() × radius     # dithered, see below
  direction, range ← normalize(aim - listener.position)

  IF collision geometry exists THEN
    # 1. Try last frame's occluder first: one triangle test instead of a tree walk.
    IF the ray hits cached_triangle within range THEN
      factor ← occlusion_scale
    ELSE
      hit ← nearest ray hit against the collision model
      IF hit THEN
        cached_triangle ← the hit triangle     # remember it for next frame
        factor ← occlusion_scale

  IF SOM geometry exists THEN
    # 2. Every SOM face the ray crosses multiplies in its own attenuation.
    FOR EACH hit along the ray against geom_som
      factor ← factor × (hit triangle's payload read as a real)

  RETURN factor
```

Three decisions, all load-bearing:

**Dithering the aim point.** A single ray from listener to source is a binary test, and a source
sliding behind a pillar would snap from unoccluded to occluded in one frame. Aiming at a random
point in a small sphere makes the test stochastic: near an edge it hits some frames and misses
others, and the emitter's occlusion smoothing (an approach at 1.0 per second) turns that into a
gradual transition. The radius the emitter passes is 0.2 m — small enough not to hear through walls,
large enough to soften every edge.

**The cached occluder.** An emitter remembers the triangle that blocked it last frame and tests that
one triangle before querying the tree. A sound behind a wall stays behind the same wall for many
frames, so the common case costs a single ray-triangle test.

**Two databases, two models.** The collision model gives a *binary* answer scaled by one global
constant (0.5 by default — a source behind anything is half as loud). The SOM gives a *graded*
answer, multiplying in a per-face value, and is what a level designer uses to make a curtain differ
from a concrete wall. They compose multiplicatively, so a source behind a wall *and* a curtain is
quieter than behind either.

## `occlusion_between`

**Contract** — The same SOM-only query between two arbitrary points, with the listener playing no
part. Used by gameplay code asking "how muffled would a sound at A be to a listener at B" — for
example, whether a creature should hear something through a door. Casts one ray, consults only the
SOM, ignores the collision model.

## `dispatch_events`

**Contract** — Drains the hearing queue into the game's handler, one call per queued (sound, range)
pair, then empties it. Runs once per update after every emitter has been advanced. The handler is
the AI's ear; with none set the queue would grow unboundedly, which is why emitters check for a
handler before queueing.

**Notes** — Batching into a queue and draining once, rather than calling the handler from inside the
emitter's update, is what keeps the handler out of the re-entry lock's way: the handler is game code
and will want to start sounds of its own.

## `object_relcase`

**Contract** — A game object is being destroyed; every emitter that names it as its owner drops the
reference. The emitter itself keeps playing.

**Notes** — This is the answer to "what happens when an emitter outlives its owner". A grenade's
explosion outlives the grenade, a creature's death cry outlives the creature. The sound is not
stopped and not detached from the world — it keeps its position and finishes. What it loses is the
ability to name an owner, which has exactly one consequence: it stops announcing itself to the AI,
because an announcement without an owner has no one for a creature to react *to*. A rebuild with a
weak-reference facility expresses this directly; the requirement is that the sound survives and the
AI link does not.

## `stop_emitters` / `pause_emitters`

**Contract** — `stop_emitters` stops every emitter immediately. `pause_emitters` moves the pause
depth by one and tells every emitter, passing the depth so each can record or match it; returns the
new depth. The release passes `depth + 1` because an emitter records the depth *at which it was
paused*, which is one above the depth after the release.

## `set_user_env`

**Contract** — Overrides the whole region system with one preset, or clears the override. Forces the
listener's blend to re-evaluate. This is the script hook for "make this cutscene sound like a
cathedral" regardless of where the listener stands.

## `set_environment` / `set_environment_size`

**Contract** — Present and inert. They were the interface to a vendor reverb extension that
configured a preset by numeric identifier and queried back its derived parameters. Nothing calls
them meaningfully any more. A rebuild should omit them; they are recorded here only because they are
part of the declared scene interface that the editor links against.
