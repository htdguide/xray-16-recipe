# src/Layers/xrRender/dxRainRender.cpp

> Advances and draws the rain drop population as camera-facing streak quads, then draws the splash particles as instanced copies of one small mesh.

**Needs** — [`Include/xrRender/RainRender.h`](../../Include/xrRender/RainRender.h.md) · [`Include/xrRender/RenderDetailModel.h`](../../Include/xrRender/RenderDetailModel.h.md) · [`xrEngine/Rain.h`](../../xrEngine/Rain.h.md) · [`xrEngine/IGame_Persistent.h`](../../xrEngine/IGame_Persistent.h.md) · [`R_DStreams.h`](R_DStreams.h.md) · [`R_Backend.h`](R_Backend.h.md) · [`FVF.h`](FVF.h.md) · [`Shader.h`](Shader.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`dx11RainBlender.cpp`](blenders/dx11RainBlender.cpp.md) · [`dxRainRender.h`](dxRainRender.h.md)
**Tier floor** — T1: it writes vertices field by field into a mapped device buffer whose stride comes from a vertex-format declaration, and it decides how much to write before it knows how much it will keep.

## Purpose

The engine's rain effect owns the *state* of a drop population — where each drop is, when it dies, where it will hit — and the queries that need the collision database. This file owns everything that needs the device: how many drops to carry at the current weather intensity, how a drop becomes four vertices, how the population is recycled as the camera moves, and how splashes are batched.

The split is not arbitrary. The engine half must exist without a renderer (a dedicated server has weather and no device); this half must exist once per graphics backend. What makes the file interesting is that the *simulation step is here, not there*: the per-drop integration, the wrap-around recycling and the death test all run inside the draw, because they are fused with the vertex write to walk the population once.

## State

```text
RECORD RainRenderer
  streak_material   : Material          # "effects/rain" over texture "fx/fx_rain"
  streak_geometry   : GeometryDecl      # position + colour + one uv, drawn from the
                                        # shared dynamic vertex stream against the
                                        # shared quad index buffer
  splash_model      : DetailModel       # loaded from "$game_meshes$/dm/rain.dm"
  splash_geometry   : GeometryDecl      # position + colour + one uv, shared dynamic
                                        # vertex AND index streams
```

Invariants:

- `splash_model` is owned: it is created through the renderer's detail-model loader at construction and destroyed through the same at teardown. Its bounding sphere is what the engine asks for when it needs a drop's size, so the model must be loaded before the first frame the engine queries.
- `streak_geometry` draws from the **shared quad index buffer** — a static index buffer holding the pattern `0,1,2 / 3,2,1` repeated, so four consecutive vertices become two triangles with no index writing. Every quad batch in the renderer relies on this; it is why the streak vertex order below is `trail-left, trail-right, head-left, head-right` and not a ring.

## Tuning constants

These are duplicated in the engine-side rain simulation and **must agree between the two halves**, because the engine spawns and retires drops with its own copy while this file integrates them with these:

```text
max_desired_items   = 2500       # population at full intensity
source_radius       = 12.5       # drops live in a cylinder of this radius around the camera
source_offset       = 40.0       # the spawn plane sits this far above the camera
max_distance        = 50.0       # = source_offset * 1.25; the longest fall considered
sink_offset         = -10.0      # = -(max_distance - source_offset); below this, respawn
drop_length         =  5.0       # streak length at full intensity
drop_width          =  0.30      # half-width of the streak quad
particles_cache     =  400       # splash meshes per draw batch
particles_time      =  0.3       # splash lifetime in seconds; also its shrink clock
```

**Notes** — the duplication is a real hazard and the source says so in a comment. A rebuild should make the constant set one shared record read by both halves. `drop_angle`, `drop_max_angle`, `drop_max_wind_vel`, `drop_speed_min`, `drop_speed_max` and `max_particles` are declared here and never read in this file — they belong to the engine half and survive only as copies.

## `render`

**Contract** — takes the engine's rain effect object. Reads the current weather's rain density and colour. Returns immediately at zero density. Otherwise: grows the population to the density-derived target, integrates every drop, emits visible ones as streak quads into the shared dynamic vertex stream, issues one draw, then walks the splash list emitting one transformed copy of the splash model per live splash in fixed-size batches. Mutates the engine's drop and splash lists (drops are reborn, splashes are freed when expired). Allocates only from the dynamic streams. Not thread-safe: it owns the shared vertex and index streams for its duration.

**Invariants** — the vertex stream is locked for exactly `target_count * 4` vertices and unlocked with however many were actually written; the unwritten tail is simply not drawn. This is the whole reason culling can be done *inside* the fill loop rather than in a pre-pass.

```text
FUNCTION render(rain)
  density = weather.rain_density
  IF density < epsilon THEN RETURN

  target = floor(0.5 * (1 + density) * max_desired_items)
  WHILE rain.drops.count < target
    rain.spawn_drop(source_radius)        # engine decides where

  visual_factor = density / 2 + 0.5       # streaks get shorter and fainter in light rain
  colour = weather.rain_colour with alpha = visual_factor

  # the plane drops are re-seeded onto, always source_offset above the camera
  spawn_plane = plane through (camera + up * source_offset) facing down

  verts = lock_vertices(target * 4, streak_geometry.stride)
  FOR EACH drop IN first target drops
    integrate_and_recycle(drop, spawn_plane)      # named step, below
    head  = drop.position
    trail = head - drop.direction * drop_length * visual_factor
    IF NOT view_frustum.intersects_sphere(midpoint(head, trail), half_length) THEN CONTINUE
    emit_streak_quad(verts, head, trail, colour, drop.uv_set)
  written = verts.count
  unlock_vertices(written)

  IF written > 0
    set_cull_mode(none)                   # a streak quad is seen from either face
    set_world_transform(identity)
    set_material(streak_material)
    set_geometry(streak_geometry)
    draw_triangles(written vertices, written/2 triangles)
    set_cull_mode(back)

  render_splashes(rain, colour)
```

### `integrate_and_recycle` — the load-bearing step

A drop is not respawned when it leaves the view; it is respawned when it leaves a **cylinder of `source_radius` around the camera**. As the camera moves, drops behind it are teleported to the opposite side of that cylinder and given a fall length that matches the geometry they would have fallen through — so a player walking forward never sees drops popping into existence ahead.

```text
FUNCTION integrate_and_recycle(drop, spawn_plane)
  IF now > drop.hit_time  THEN rain.on_hit(drop.hit_point)   # spawns splash + wallmark
  IF now > drop.life_end  THEN rain.respawn(drop, source_radius)

  drop.position = drop.position + drop.direction * drop.speed * frame_delta

  horizontal = (drop.position - camera) projected onto the ground plane
  IF horizontal.length <= source_radius + 0.5 THEN RETURN      # still inside, nothing to do

  IF drop.position.height - camera.height < sink_offset
    invalidate(drop)                      # fell too far below; let the engine respawn it
    RETURN

  # mirror it across the cylinder: move it back through the camera by its own
  # horizontal distance plus a radius, so it lands on the far rim
  drop.position = drop.position - normalize(horizontal) * (horizontal.length + source_radius)

  IF NOT spawn_plane.intersect_ray(drop.position, -drop.direction) -> seed_point
    invalidate(drop)                      # cannot be traced back to the spawn plane
    RETURN

  fall_from_seed = distance(drop.position, seed_point)
  height = max_distance
  IF collision_ray(seed_point, drop.direction, height) hits
    IF height^2 <= fall_from_seed^2
      invalidate(drop)                    # it would already have hit; give up on it
    ELSE
      rain.renew(drop, height - fall_from_seed, hits_geometry = true)
  ELSE
    rain.renew(drop, max_distance - fall_from_seed, hits_geometry = false)
```

**Notes** — the comparison `height² ≤ fall_from_seed²` avoids a square root on the reject path; the accept path pays for one. The collision query is `both faces` so a drop entering a building from above still terminates on the roof's back face.

### `emit_streak_quad`

A streak is a screen-facing ribbon, not a billboard: its long axis is the drop's own direction, and only its *width* faces the camera.

```text
FUNCTION emit_streak_quad(verts, head, trail, colour, uv_set)
  axis     = normalize(head - trail)
  centre   = midpoint(head, trail)
  to_cam   = normalize(centre - camera)
  widthdir = cross(to_cam, axis)          # perpendicular to both: the ribbon's width
  w = drop_width
  push(trail - widthdir*w, colour, UV[uv_set][0])
  push(trail + widthdir*w, colour, UV[uv_set][1])
  push(head  - widthdir*w, colour, UV[uv_set][2])
  push(head  + widthdir*w, colour, UV[uv_set][3])
```

Two texture-coordinate sets exist and each drop carries which one it uses:

```text
UV[0] = (0,1) (0,0) (1,1) (1,0)
UV[1] = (1,0) (1,1) (0,0) (0,1)
```

Set 1 is set 0 mirrored in both axes. **This is why the rain texture does not visibly tile**: with one set, 2500 identical quads show the same streak image and read as a repeating pattern; flipping half of them at spawn breaks it for free.

### `render_splashes`

Splashes are copies of one small mesh, each with its own transform and a uniform shrink over its life. They are batched because a splash is a handful of triangles and a draw per splash would be the dominant cost.

```text
FUNCTION render_splashes(rain, colour)
  IF rain.splashes is empty THEN RETURN
  set_material(splash_model.material)
  vcap = particles_cache * splash_model.vertex_count
  icap = particles_cache * splash_model.index_count
  verts, idx = lock(vcap, icap)
  count = 0
  FOR EACH splash IN rain.splashes            # singly linked, walked once
    splash.time = splash.time - frame_delta
    IF splash.time < 0
      rain.free(splash)                       # unlink and return to the pool
      CONTINUE
    IF NOT view_frustum.intersects_sphere(splash.bounds)
      CONTINUE
    scale = splash.time / particles_time      # shrinks to nothing as it expires
    transform = splash.transform * uniform_scale(scale)
    splash_model.emit(transform, verts, colour, idx, base = count * vertex_count)
    advance verts, idx; count = count + 1
    IF count = particles_cache
      unlock(vcap, icap); set_geometry(splash_geometry)
      draw_triangles(vcap vertices, icap/3 triangles)
      verts, idx = lock(vcap, icap); count = 0
  unlock(count * vertex_count, count * index_count)
  IF count > 0
    set_geometry(splash_geometry); draw_triangles(...)
```

**Invariants** — the lock is always taken for the *full* batch capacity and released with the actual count, and a lock is always matched by an unlock even when `count` is zero, because the stream's write cursor must be advanced past the reservation either way. Index data is emitted as well as vertices, with a per-batch base offset added to every index, because the batch is one draw over one buffer region.

**Notes** — the splash mesh is loaded from the shipped detail-model file `dm/rain.dm`; the format is frozen. The detail model's own `emit` writes both vertices and indices and applies a colour and an optional texture-frame shift — the rain path uses colour only.

## `drop_bounds`

**Contract** — returns the splash model's bounding sphere. The engine uses it to size the wallmark and the collision query for a hit, so it must be the *model's* sphere and not the streak's.

## `copy`

**Contract** — replaces this renderer's whole state with another instance's, including the owned splash model reference. Used when the weather system rebuilds its effect objects without wanting to reload the model.

**Notes** — the model pointer is copied, not duplicated, so exactly one of the two instances may destroy it. In the original this is a raw assignment with no ownership transfer recorded, which is a latent double-free; a rebuild should move rather than copy, or make the model a shared immutable resource.
