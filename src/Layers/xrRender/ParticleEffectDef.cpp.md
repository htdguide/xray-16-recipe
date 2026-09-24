# src/Layers/xrRender/ParticleEffectDef.cpp

> The authored definition of one particle effect: its frozen on-disk record, the texture-atlas frame convention, and the two per-step behaviours (frame animation and world collision) that the renderer adds on top of the simulator's own action list.

**Needs** — [`ParticleEffectDef.h`](ParticleEffectDef.h.md) · [`ParticleEffect.h`](ParticleEffect.h.md) · [`Shader.h`](Shader.h.md) · [`xrParticles/psystem.h`](../../xrParticles/psystem.h.md) · [`xrParticles/particle_actions.h`](../../xrParticles/particle_actions.h.md) · [Seam: Static collision database](../../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — [`ParticleEffectDef.h`](ParticleEffectDef.h.md)
**Tier floor** — T1: the definition record is read from the shipped particle library as a byte image (the frame block is copied verbatim, not field by field), and the action list is copied as an opaque byte run that the simulator later re-parses.

## Purpose

An **effect definition** is the authored description of one particle emitter: how many particles it may have, which material draws them, how its texture atlas is cut into animation frames, whether it collides with the world, and — as an opaque byte block — the list of simulation actions that chapter 12 executes. It is the *class*; [`ParticleEffect.cpp`](ParticleEffect.cpp.md) is the *instance*.

The split matters because a definition is shared: one definition is loaded once into the library and every playing copy of that effect points at it, including the one compiled material it owns. Nothing in a definition is per-instance mutable state.

Two behaviours live here rather than in the simulator because both need renderer-side knowledge the simulator does not have: frame animation needs the atlas layout, and collision needs the level's collision database and the effect's ownership so that a particle can be removed.

## State

```text
RECORD FrameLayout            # copied to and from the library file as a raw 28-byte image
  tex_size    : pair of real  # ONE frame's size in normalized texture coordinates
  reserved    : pair of real  # written and read, never used; must round-trip
  frame_dim_x : int (32-bit)  # frames per row of the atlas
  frame_count : int (32-bit)  # total frames in the atlas
  speed       : real          # frames per second when animated
```

```text
RECORD EffectDefinition
  name              : text          # library key; for the loose-file form it is path + basename, no extension
  flags             : int (32-bit)  # the df* set below
  shader_name       : text          # material template name, resolved against the material library
  texture_name      : text          # texture handed to that template
  cached_material   : material      # compiled once when the library loads, shared by every instance
  frame             : FrameLayout
  actions           : bytes         # compiled action list, opaque here, parsed by the simulator
  time_limit        : real          # seconds; meaningful only when the time-limit flag is set
  max_particles     : int
  velocity_scale    : vector        # x and y add to the sprite half-extents per unit of speed
  align_default_rot : vector        # euler angles, default (-quarter turn, 0, 0)
  collide_one_minus_friction : real # stored pre-subtracted: 1 - friction
  collide_resilience         : real
  collide_sqr_cutoff         : real # stored pre-squared: cutoff * cutoff
```

**The flag set** — these bit positions are in the shipped files and are frozen:

```text
ENUM DefinitionFlag
  sprite            = bit 0    # this effect draws as camera- or path-oriented quads
  # bit 1 was an object-instancing mode; no shipped effect uses it
  framed            = bit 10   # the texture is an atlas, cut by FrameLayout
  animated          = bit 11   # the frame index advances with time
  random_frame      = bit 12   # a newborn particle starts on a random frame
  random_playback   = bit 13   # half of newborn particles animate backwards
  time_limit        = bit 14   # the effect stops itself after time_limit seconds
  align_to_path     = bit 15   # orient the quad by velocity instead of by the camera
  collision         = bit 16   # sweep each particle against the world each step
  collision_del     = bit 17   # ... and kill it on first contact instead of bouncing
  velocity_scale    = bit 18   # stretch the quad with speed
  collision_dyn     = bit 19   # collide against dynamic objects too, not only static geometry
  world_align       = bit 20   # when path-aligned and nearly still, use align_default_rot
  face_align        = bit 21   # when path-aligned and moving, face the quad along velocity
  culling           = bit 22   # do not disable backface culling for this effect
  cull_ccw          = bit 23   # ... and cull the counter-clockwise face rather than the clockwise one
```

**Invariants**

- The frame layout is written to and read from the library as one raw image of its five fields, `reserved` included. A rebuild that parses field by field must still consume and re-emit those two unused reals, because the byte offsets of everything after them are fixed by the shipped file.
- `tex_size` is the *frame's* size in normalized coordinates, not the texture's — the authoring tool divided frame pixels by texture pixels before storing. The default is a 32×64 frame in a 256×128 texture.
- `frame_count` must not exceed `frame_dim_x × (rows the atlas actually has)`; nothing checks it, and an over-large count walks off the bottom of the atlas into whatever the sampler's address mode returns.
- The animated frame index is stored on each particle as a fixed-point value scaled by **255**, not 256. Both the writer (`floor(f × 255)`) and the reader (`floor(index / 255)`) use 255, so the pair is consistent; a rebuild that picks 256 for either half drifts. Because the particle's field is 16 bits, `frame_count` above 257 overflows it.
- Friction and cutoff are stored **already transformed** — one-minus-friction and cutoff-squared. This is a file format decision, not an optimization to re-derive: the shipped numbers are in that form.
- The material is created once for the whole definition when the library loads and destroyed once when it unloads. An instance never creates its own; it copies the reference. So a definition whose material failed to compile yields instances that silently draw nothing rather than instances that retry.
- Loading tolerates a missing align-to-path chunk (the default rotation stands) but requires every other chunk its flags claim. A flag set without its chunk is a corrupt file and aborts.

## `load` — read a definition from the library

**Contract** — reads one definition out of a chunked reader. Fails (without side effects worth unwinding) if the version word does not match exactly; aborts if a chunk the flags promise is absent. Allocates the action-list copy. The action bytes are *copied*, not referenced, because the library reader is closed before any instance is compiled.

```text
FUNCTION load(reader) -> bool
  require chunk VERSION;  IF version != 1 THEN RETURN false
  require chunk NAME;          name = read_string
  require chunk EFFECTDATA;    max_particles = read_int32
  require chunk ACTIONLIST;    actions = copy of the whole chunk payload
  read chunk FLAGS into flags

  IF flags has sprite THEN
    require chunk SPRITE;      shader_name = read_string; texture_name = read_string
  IF flags has framed THEN
    require chunk FRAME;       frame = read raw image, 28 bytes
  IF flags has time_limit THEN
    require chunk TIMELIMIT;   time_limit = read_real
  IF flags has collision THEN
    require chunk COLLISION;   one_minus_friction, resilience, sqr_cutoff = three reals
  IF flags has velocity_scale THEN
    require chunk VEL_SCALE;   velocity_scale = three reals
  IF flags has align_to_path AND chunk ALIGN_TO_PATH present THEN
    align_default_rot = three reals        # absent is legal: keep the default
  RETURN true
```

**Notes** — The chunk identifiers are frozen: version 0x0001, name 0x0002, effect data 0x0003, action list 0x0004, flags 0x0005, frame 0x0006, sprite 0x0007, time limit 0x0008, collision 0x0021, velocity scale 0x0022, align-to-path 0x0025. Two identifiers are reserved and must not be reused: 0x0009 (a second time-limit chunk that no shipped file carries) and 0x0020 (obsolete action *source text*, dropped when actions moved to a compiled form). 0x0024 carries the uncompiled, editable action list and is written only by the authoring build; the game build skips it, so a rebuild that only plays the game can ignore it and must still not trip over it.

The action-list chunk is the one place a definition stores a nested format it does not understand: its first word is the count of enabled actions, and the rest is each action's type code followed by that action's own parameters. Chapter 12 owns that grammar.

## `load_from_config` — read a definition from a loose text file

**Contract** — the same record read out of the configuration format instead of the binary library, used when the engine is pointed at a directory of individual effect files rather than a packed library. Section and key names are frozen because those files ship in some distributions.

```text
FUNCTION load_from_config(config) -> bool
  max_particles = config["_effect"]["max_particles"]
  flags         = config["_effect"]["flags"]                 # same bit positions as the binary form
  IF sprite        THEN shader_name, texture_name FROM section "sprite", keys "shader","texture"
  IF framed        THEN frame FROM section "frame": tex_size, reserved, dim_x, frame_count, speed
  IF time_limit    THEN time_limit FROM "timelimit"/"value"
  IF collision     THEN FROM "collision": one_minus_friction, collide_resilence, collide_sqr_cutoff
  IF velocity_scale THEN FROM "velocity_scale"/"value"
  IF align_to_path THEN FROM "align_to_path"/"default_rotation"
  RETURN true
```

**Notes** — The key `collide_resilence` is misspelled in the shipped files and in the writer. It is frozen; a rebuild must read and write that spelling. The `version` key is written but never read back — the loose form is gated by nothing, so a rebuild cannot detect an old one.

The editable action list, when present, lives in sections named `action_0000`, `action_0001`, … with a zero-padded four-digit index and a numeric `action_type` key, counted by `_effect`/`action_count`.

## `save`, `save_to_config`

**Contract** — write the record back in either form, emitting exactly the chunks (or sections) the flags claim. Symmetric with the loaders in every field, including the two reserved reals of the frame layout. A definition is only written by the authoring path; the shipping engine never writes one.

## `execute_animate` — advance the atlas frame

**Contract** — steps every particle's frame index by the definition's speed, wrapping in whichever direction that particle was born with. Pure over the particle array; no allocation; called once per fixed simulation step, and only when the effect is **both** framed and animated.

```text
FUNCTION execute_animate(particles, dt)
  step = frame.speed * dt
  FOR EACH p IN particles
    f = p.frame / 255 + (p.animates_backwards ? -step : +step)
    IF f > frame.frame_count THEN f = f - frame.frame_count    # wrap forward
    IF f < 0                 THEN f = f + frame.frame_count    # wrap backward
    p.frame = floor(f * 255)
```

**Notes** — The forward wrap compares against `frame_count` rather than `frame_count - 1`, so the index can momentarily sit exactly on `frame_count` before the next step pulls it back; the atlas lookup floors it, and the resulting one-frame overshoot is what the shipped art was authored against. Leave it.

## `execute_collision` — sweep particles against the world

**Contract** — for each particle, casts the segment from its previous position to its current one against the level; on a hit either removes the particle or reflects its velocity and re-integrates the remainder of the step. Calls the owner's collision callback on the first hit of each particle only, and abandons that particle's collision if the callback declines. Queries the collision database (static only, or static plus dynamic objects when the dynamic flag is set). Does not allocate. Runs inside the fixed step, so it must be cheap: at most two picks per particle per step.

**Invariants**

- Particles are traversed **from the end of the array backwards**, because removal swaps the last element into the removed slot. A forward walk would skip the swapped-in particle.
- A particle whose travel this step is below epsilon is snapped back to its previous position rather than tested. This is what keeps a stationary particle resting on a surface instead of jittering through it.
- The resolve loop runs at most twice. A particle that would need a third resolve keeps the velocity from the second and is allowed to end the step inside geometry; the alternative is an unbounded loop in a corner.

```text
FUNCTION execute_collision(particles, dt, owner, on_contact)
  FOR i FROM last DOWNTO first
    p = particles[i]
    picks = 0
    REPEAT
      again = false
      travel = p.pos - p.prev_pos
      dist = length(travel)
      IF dist < epsilon THEN
        p.pos = p.prev_pos                      # resting contact: do not test, do not drift
      ELSE
        dir = travel / dist
        IF world.ray_pick(p.prev_pos, dir, dist) HITS THEN
          point  = p.prev_pos + dir * hit.range
          normal = hit is a dynamic object ? straight up : face normal of the hit triangle
          picks = picks + 1
          IF picks == 1 AND on_contact is set AND NOT on_contact(owner, p, point, normal) THEN BREAK
          IF flags has collision_del THEN
            remove particle i from the effect
          ELSE
            vn = normal * dot(p.vel, normal)    # normal component
            vt = p.vel - vn                     # tangential component
            IF length_squared(vt) <= collide_sqr_cutoff THEN
              p.vel = vt - vn * collide_resilience              # too slow to rub: no friction
            ELSE
              p.vel = vt * collide_one_minus_friction - vn * collide_resilience
            p.pos = p.prev_pos + p.vel * dt     # re-integrate the whole step from the old position
            again = true
    UNTIL NOT again OR picks >= 2
```

**Notes** — Two details are load-bearing and look arbitrary. First, a hit on a *dynamic* object reports a straight-up normal rather than the object's surface normal: the dynamic query returns an object, not a triangle, and the engine does not ask the object for a contact normal. Sparks landing on a moving crate therefore bounce upward regardless of the crate's facing — reproduce it, because tuning was done against it. Second, the re-integration restarts from the *previous* position with the full step, not from the contact point with the remaining step; a bounced particle therefore travels slightly further than a physically exact sweep would put it.

The friction cutoff is a *tangential speed* threshold: below it the surface is treated as perfectly slippery so that slow particles slide out of contact instead of sticking. Because the stored value is already squared, the comparison is against the squared tangential speed with no square root.

## `create_material`, `destroy_material`, `set_name`

**Contract** — compile the definition's material from its shader and texture names, release it, and rename the definition. The material is only created when both names are present; a definition with neither is a non-drawing effect (it exists to run actions and spawn children).
