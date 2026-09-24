# src/Layers/xrRender/ParticleEffect.cpp

> One playing instance of a particle effect: it drives the simulator at a fixed 33 ms step, keeps its own bounding volume current for the visibility walk, and turns the resulting particle array into oriented textured quads in the dynamic vertex stream.

**Needs** — [`ParticleEffect.h`](ParticleEffect.h.md) · [`ParticleEffectDef.h`](ParticleEffectDef.h.md) · [`FBasicVisual.h`](FBasicVisual.h.md) · [`dxParticleCustom.h`](dxParticleCustom.h.md) · [`R_Backend.h`](R_Backend.h.md) · [`R_DStreams.h`](R_DStreams.h.md) · [`FVF.h`](FVF.h.md) · [`Include/xrRender/ParticleCustom.h`](../../Include/xrRender/ParticleCustom.h.md) · [`xrParticles/psystem.h`](../../xrParticles/psystem.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`ParticleEffect.h`](ParticleEffect.h.md)
**Tier floor** — T1: it writes vertices directly into a mapped device buffer at a fixed stride, and the sprite builder is written twice — once in 4-wide float operations and once scalar — because it is the frame's hottest inner loop.

## Purpose

A **particle effect** is a renderable visual (it takes part in the visibility walk and enters the draw stream like a mesh) whose geometry is generated every frame from a simulated particle array. This file is the whole instance: playback state, the fixed-step pump, the bounding volume the visibility pass culls against, and the quad builder.

The division of labour is worth stating once. Chapter 12 owns *what the particles do* — it holds the array, runs the action list, spawns and retires particles. [`ParticleEffectDef.cpp`](ParticleEffectDef.cpp.md) owns *what the effect is* — the authored record. This file owns *when the simulation advances* and *what the particles look like*, and it is the only one of the three that touches the graphics device.

## State

```text
RECORD ParticleEffect                     # is-a renderable visual, is-a particle-custom visual
  definition        : EffectDefinition or none
  effect_handle     : int     # identifies the particle array inside the simulator
  action_list_handle: int     # identifies the compiled action list inside the simulator
  elapsed_limit     : real    # seconds remaining before a time-limited effect stops itself
  accumulated_ms    : int     # unspent real time, carried between frames
  initial_position  : vector  # where the effect sits while it is not playing
  transform         : matrix  # parent transform, applied at draw time in "transform" mode
  runtime_flags     : int (8-bit)
      playing        = bit 0
      deferred_stop  = bit 1  # emission stopped; live particles still finish
      transform_mode = bit 2  # see below
      hud_mode       = bit 3  # drawn through the weapon-view projection
  on_destroy        : callback or none
  on_collision      : callback or none
  geometry, material                      # inherited from the renderable-visual base
  visibility_box, visibility_sphere       # inherited; recomputed every step
```

**Invariants**

- The two simulator handles are allocated in the constructor and released in the destructor, once each. They are the instance's identity to chapter 12; nothing else identifies it.
- **The parent transform has two mutually exclusive modes, and the shipped effects depend on both.** In *moving* mode (the default), the parent's position and velocity are pushed *into* the simulator — the emitter itself moves through the world and particles, once born, are left behind in world space. In *transform* mode, the simulator runs in the parent's local space and the whole cloud is transformed at draw time, so the particles follow the parent rigidly. An effect attached to a weapon muzzle wants transform mode; smoke rising from a fire wants moving mode.
- The bounding volume is recomputed from the live particles at the end of each fixed step: the box over every particle position, grown by the largest single size component found among them, and the sphere derived from that box. When the effect is not playing, the volume collapses to a point at the initial position grown by a hair. A rebuild must keep updating it while stopping-but-still-draining, or the last puff of a deferred stop pops out of view.
- The box is grown by the largest size component of *any* particle, applied uniformly — not per particle. It is a cheap over-estimate; halving it to be tight would let large sprites at the cloud's edge get culled while still visible.
- `deferred_stop` and `playing` clear together, and only when the particle count reaches zero.

## The fixed step

**`uDT_STEP` is 33 milliseconds and `fDT_STEP` is that in seconds.** Particles are simulated at ~30 Hz regardless of frame rate. Real elapsed time accumulates and is spent in whole steps; the remainder carries.

**At most three steps are spent in one frame.** Above that the surplus is dropped. The reason is stated in the source and is worth preserving: after a level load or a hitch, the accumulated time would otherwise be paid in one burst of dozens of steps across every playing effect at once. Three steps is 99 ms — an effect only *loses* time below about 10 frames per second, which is already unplayable. The consequence a rebuild inherits: particle simulation is not frame-rate independent at the bottom end, deliberately.

## `on_frame` — advance the simulation

**Contract** — takes the frame's elapsed milliseconds, spends what it can in fixed steps, runs the simulator plus the definition's own two behaviours per step, and leaves the bounding volume current. Does not touch the device. Returns nothing; failure is not a concept here.

```text
FUNCTION on_frame(frame_ms)
  IF definition is none OR NOT playing THEN
    visibility volume = point at initial_position, grown by epsilon
    RETURN

  accumulated_ms = accumulated_ms + frame_ms
  steps = 0
  IF accumulated_ms >= 33 THEN
    steps = accumulated_ms / 33
    accumulated_ms = accumulated_ms MOD 33
    clamp steps to [0, 3]                     # see "The fixed step" above

  WHILE steps > 0
    steps = steps - 1
    IF definition has time_limit AND NOT deferred_stop THEN
      elapsed_limit = elapsed_limit - step_seconds
      IF elapsed_limit < 0 THEN
        elapsed_limit = definition.time_limit  # rearmed, so a replay starts full
        stop(deferred = true)
        BREAK

    simulator.update(effect_handle, action_list_handle, step_seconds)
    particles = simulator.particles(effect_handle)

    IF definition is framed AND animated THEN definition.execute_animate(particles, step_seconds)
    IF definition has collision       THEN definition.execute_collision(particles, step_seconds, self, on_collision)

    IF particles is non-empty THEN
      box = bounds over every particle position
      grow box by the largest size component seen
      visibility volume = box and its sphere

    IF deferred_stop AND particles is empty THEN
      clear playing and deferred_stop
      BREAK
```

**Notes** — The frame-animation test requires **both** the framed and animated flags; a framed effect that is not animated keeps whatever frame each particle was born with, which is how random-frame-but-static sprites (debris, sparks) are authored.

The time limit is re-armed rather than zeroed when it fires, so a stopped effect that is played again runs a full duration without an explicit reset.

## `render` — build and submit the quads

**Contract** — locks a run of the shared dynamic vertex stream, writes four vertices per live particle, unlocks the exact count written, and issues one indexed triangle-list draw through the shared quad index buffer. Does nothing when the effect has no particles or is not a sprite effect. Sets and restores the cull mode; in weapon-view mode it also swaps the projection for the duration of the draw. Runs on a render command list; the vertex stream lock is the serializing resource.

**Invariants**

- Four vertices per particle, two triangles, using the renderer's **shared quad index buffer** — the effect contributes no indices of its own. That buffer is built once for a fixed number of quads, which bounds how many particles one effect may draw in one call; an effect whose budget exceeds it indexes past the end.
- The draw is a plain triangle list, `particle_count × 4` vertices and `particle_count × 2` primitives, at the offset the stream lock returned.
- Cull mode is explicitly restored to counter-clockwise afterwards, because the backend's cached state is shared across draws and particles are the only visuals that routinely turn culling off.

```text
FUNCTION render(cmd_list)
  IF the backend is the one that pays for distance culling THEN
    IF squared distance from camera to initial_position > (100 * visibility_distance)^2 THEN RETURN

  particles = simulator.particles(effect_handle)
  IF particles is empty OR definition is not a sprite effect THEN RETURN

  vertices, offset = dynamic_vertex_stream.lock(particle_count * 16, stride)
  build_sprites(vertices, particles)               # see below
  count = particle_count * 4
  dynamic_vertex_stream.unlock(count, stride)

  IF hud_mode THEN
    save projection and full transform
    build a projection from the weapon-view field of view, the frame's aspect,
      the near plane reserved for weapon geometry, and the weather's far plane
    push it, switch the backend to its near-depth range, refresh the projected-texture matrix

  cmd_list.set_world_transform(identity)           # particles are already in world space
  cmd_list.set_geometry(quad geometry)
  cmd_list.set_cull_mode(definition has culling
                          ? (cull_ccw ? counter-clockwise : clockwise)
                          : none)
  cmd_list.draw_indexed(triangle list, offset, count vertices, count/2 triangles)
  cmd_list.set_cull_mode(counter-clockwise)

  IF hud_mode THEN restore the backend's normal depth range, projection and transform
```

**Notes** — The distance cut at a hundred times the visibility-distance console setting exists only on the backend where submitting the draw is expensive enough that culling beats drawing; the other backend draws it. That asymmetry is a performance patch, not a behavioural rule — a rebuild may apply it to all backends or none, at the cost of matching one backend's visible particle range exactly.

The lock asks for **four times** the vertices actually written (sixteen per particle, not four) and then unlocks the true count. Nothing in the source explains the factor; it reads as defensive headroom from an era when a particle could expand to more than one quad. A rebuild that locks exactly what it writes is correct, provided the stream's lock honours the request it was given.

Weapon-view mode exists because the weapon is rendered with its own narrow field of view and a near plane much closer than the world's; a muzzle-flash effect must share that projection or it will be clipped or visibly wrong in scale. The projected-texture matrix has to be rebuilt alongside it, since deferred lighting samples through it.

## `build_sprites` — orientation, atlas frame and vertex layout

**Contract** — writes four vertices per particle into the locked buffer. Pure with respect to the particle array. This is the hottest loop in the file; it exists in a 4-wide-float form and a scalar fallback, which must agree bit-for-bit closely enough that no visible difference appears between platforms.

Each vertex is `(position, packed colour, texture coordinate pair)` at a fixed stride — a lit, textured, untransformed vertex. Particles are emitted already in world space (the world transform is identity at draw time) except in transform mode, where each position and direction is transformed by the parent first.

**The quad is built from two axes and two half-extents**, rotated in its own plane by the particle's roll:

```text
FUNCTION emit_quad(out, up_axis T, right_axis R, centre, uv_lt, uv_rb, half_w, half_h, colour, sin_roll, cos_roll)
  across = (T * sin_roll + R * cos_roll) * half_w
  along  = (T * cos_roll - R * sin_roll) * half_h
  a = along - across
  b = along + across
  # the four corners, in the order the shared quad index buffer expects
  out[0] = centre - b, colour, (uv_lt.x, uv_rb.y)
  out[1] = centre + a, colour, (uv_lt.x, uv_lt.y)
  out[2] = centre - a, colour, (uv_rb.x, uv_rb.y)
  out[3] = centre + b, colour, (uv_rb.x, uv_lt.y)
```

**The half-extents** start at half the particle's size in x and y. When the velocity-scale flag is set, the particle's speed times the definition's scale is *added* to each — that is what makes a fast spark a long streak and a slow one a dot.

**Orientation is chosen per particle, in this order:**

```text
IF NOT align_to_path THEN
  T = camera up, R = camera right                     # a plain screen-facing billboard
ELSE IF speed < epsilon AND world_align THEN
  M = rotation from the definition's default euler angles
  T = M.forward, R = M.right                          # a still particle pinned to an authored plane
ELSE IF speed >= epsilon AND face_align THEN
  M = orthonormal basis whose forward is the velocity direction
      (up is world up, or world forward when velocity is within ~8 degrees of up)
  T = M.up, R = M.right                               # the quad's plane is perpendicular to travel
ELSE
  dir = velocity direction, or the heading/pitch of the default rotation when nearly still
  R   = normalize(cross(dir, camera forward)); T = dir
                                                      # stretched along travel, edge-on to the camera
```

In transform mode each branch also transforms the centre by the parent matrix, and the basis or direction with it, before emitting.

**The atlas frame** is the particle's fixed-point frame index floored to an integer and looked up in the definition's frame layout; an unframed effect uses the whole texture, corner coordinates `(0,0)` to `(1,1)`.

**Notes** — Two details are easy to lose. First, the near-degenerate guard in the face-aligned basis triggers when the velocity is within about eight degrees of world up (a dot product above 0.99), at which point the seed axis switches from world up to world forward; without it the cross product collapses and the quad vanishes. Second, the roll's sine and cosine are **cached across consecutive particles with the same roll** and recomputed only when it changes, with a sentinel initial value that cannot equal any real roll so the first particle always computes. Most effects give every particle the same roll, so this removes two transcendental calls per particle; a rebuild that computes them unconditionally is correct but measurably slower on large clouds.

Particle rendering was tried as a parallel loop over the array and is deliberately **single-threaded**: the source records that the parallel version was slower, the per-particle work being too small to cover the scheduling. A rebuild should start single-threaded.

## `compile` — bind an instance to a definition

**Contract** — points the instance at a definition, rebuilds its device resources, hands the definition's compiled action bytes and particle budget to the simulator, installs the birth and death callbacks, arms the time limit, and adopts the definition's shared material. Must be called before the effect can play.

```text
FUNCTION compile(definition)
  self.definition = definition
  destroy device resources, then create them        # the material may have changed
  simulator.load_actions(action_list_handle, definition.actions)
  simulator.set_max_particles(effect_handle, definition.max_particles)
  simulator.set_callbacks(effect_handle, on_particle_birth, on_particle_death, self)
  IF definition has time_limit THEN elapsed_limit = definition.time_limit
  material = definition.cached_material
```

## `on_particle_birth`, `on_particle_death`

**Contract** — the simulator calls these as particles appear and disappear. Birth applies the two per-particle randomizations the definition asks for: a random starting atlas frame, and — for half of the particles, when random playback is set — a backwards animation direction. Death does nothing at this level; the group layer overrides both to spawn child effects.

**Notes** — The backwards direction is decided by a coin flip per particle, so a random-playback effect animates roughly half its particles in each direction. This is the cheap way to make a looping atlas stop looking like a loop.

## `play`, `stop`, `is_playing`, `set_hud_mode`, `get_time_limit`, `name`, `particle_count`

**Contract** — playback control and the properties the owning game object reads. `stop` takes a *deferred* argument, and the distinction is load-bearing: a deferred stop halts emission and lets the existing particles live out their ages (a fire being extinguished), while an immediate stop discards them (an effect whose owner was destroyed). `get_time_limit` returns a negative value for an effect with no limit, and the base interface reads that as **looped** — so "looped" is not a stored flag anywhere, it is the absence of a time limit.

## `update_parent`

**Contract** — sets the effect's placement for the coming frames. In transform mode it stores the matrix for use at draw time. In moving mode it records the origin as the fallback bounding position and pushes the matrix and the parent's velocity into the simulator, where actions that emit in a direction or inherit parent motion read them.

## `on_device_create`, `on_device_destroy`

**Contract** — acquire and release the quad geometry description (the lit-vertex format bound to the shared dynamic vertex stream and the shared quad index buffer) and the material reference. Both are no-ops for a non-sprite effect. Called on a device reset, so they must be idempotent in pairs.

## `copy`

**Contract** — duplicating a particle visual is unsupported and fails fatally. A rebuild should make it unrepresentable rather than a runtime abort: the simulator handles are not duplicable, and everything a caller could want from a copy is obtained by creating a second instance from the same definition.
