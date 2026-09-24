# src/Include/xrRender/KinematicsAnimated.h

> The animation-playing half of a skeleton: motion banks, blend slots, body partitions, channels, and the per-frame track update the whole creature layer drives.

**Needs** — [`Kinematics.h`](Kinematics.h.md) · [`animation_blend.h`](animation_blend.h.md) · [`animation_motion.h`](animation_motion.h.md) · [`xrCore/Animation/SkeletonMotions.hpp`](../../xrCore/Animation/SkeletonMotions.hpp.md) · [`Layers/xrRender/KinematicAnimatedDefs.h`](../../Layers/xrRender/KinematicAnimatedDefs.h.md) · [`Layers/xrRender/KinematicsAddBoneTransform.hpp`](../../Layers/xrRender/KinematicsAddBoneTransform.hpp.md)
**Used by** — [`Kinematics.h`](Kinematics.h.md) · [`RenderVisual.h`](RenderVisual.h.md) · [`animation_blend.h`](animation_blend.h.md) · [`SkeletonAnimated.cpp`](../../Layers/xrRender/SkeletonAnimated.cpp.md) · [`SkeletonAnimated.h`](../../Layers/xrRender/SkeletonAnimated.h.md) · [`AmebaZone.h`](../../xrGame/AmebaZone.h.md) · [`Artefact.cpp`](../../xrGame/Artefact.cpp.md) · [`CharacterPhysicsSupport.cpp`](../../xrGame/CharacterPhysicsSupport.cpp.md) · [`HairsZone.h`](../../xrGame/HairsZone.h.md) · [`HangingLamp.cpp`](../../xrGame/HangingLamp.cpp.md) · [`Helicopter.cpp`](../../xrGame/Helicopter.cpp.md) · [`PHDebug.cpp`](../../xrGame/PHDebug.cpp.md) · [`PhysicObject.cpp`](../../xrGame/PhysicObject.cpp.md) · [`Weapon.h`](../../xrGame/Weapon.h.md) · _and 28 more_
**Tier floor** — T2: a pure interface over fixed-size blend pools; the pools are preallocated and reused rather than grown, and the whole surface is called several hundred times a frame, which forbids per-call allocation but not a managed tier.

## Purpose

A *rigid* skeleton has bones that something else poses. An *animated* skeleton additionally owns a set of motion banks loaded from the model's animation files and a pool of running blends, and computes each bone's pose by mixing whatever blends currently cover it. This header is what the creature layer, the weapon layer, the vehicle layer and the script layer all speak to when they want something to move.

The split from [`Kinematics.h`](Kinematics.h.md) is not cosmetic. A door, a crate and a corpse are skeletons but never play animations; keeping animation out of the base interface keeps the blend pool — which is large, per-instance and preallocated — off every one of them.

## State

```text
RECORD AnimatedSkeleton                  # what an implementor must have
  motion_slots   : list<MotionBank>      # ordered; later slots shadow earlier ones on name lookup
  partition      : Partition             # authored grouping of bones into body parts
  blend_pool     : list<Blend>           # fixed size, preallocated; a free slot is marked, not removed
  cycles         : list<Blend ref>[PARTS] # currently running looping/one-shot motions, per body part
  effects        : list<Blend ref>       # currently running additive one-shots ("fx"), bone-local
  per_bone_blends: list<list<Blend ref>> # for each bone, the blends that cover it
  channel_factors: list<real>[CHANNELS]  # per-channel weight, settable at run time
  last_track_ms  : int                   # wall clock of the last track advance
  track_driver   : optional<UpdateTracksDriver>
  blend_destroy_listener : optional<BlendDestroyListener>

RECORD MotionBank                        # shared between every instance of the model; see SkeletonMotions
  cycles      : map<text,int>            # name -> index within this bank
  effects     : map<text,int>
  definitions : list<MotionDef>          # authored speed/power/accrue/falloff/flags per motion
  bone_tracks : map<text, list<Motion>>  # per bone, one key stream per motion

RECORD Partition                         # PARTS entries, some unnamed and therefore unused
  parts : list<(name : text, bones : list<int>)>
```

**Invariants**

- The blend pool never grows. Exhausting it is a fatal error, not a dropped animation — the engine would rather stop than silently play the wrong thing. The pool is sized `MAX_BLENDS × PARTS × CHANNELS`.
- A blend in the pool is free exactly when its curvature state is the free marker; nothing else distinguishes a free slot.
- `PARTS` is **4** and `CHANNELS` is **4**, and both numbers are baked into shipped model data: the partition table in an animation bank has four entries, and motion definitions name a channel in a two-bit field. They are not tunable.
- A part with no name is not a part. Playback on an unnamed part index is silently a no-op, which is how models with fewer than four body parts work.

## The three concepts a rebuilder needs before the methods make sense

**Body part.** The bones of a creature are partitioned into named groups — typically legs, torso, head, and one spare — by data authored alongside the animations. A cycle plays *on a part*: the legs can walk while the torso reloads. A blend started on the sentinel part index starts on every named part at once.

**Channel.** Orthogonal to parts. Each running blend declares a channel, and the channels are mixed into the final pose by a fixed rule table: **channels 0 and 1 mix by interpolation, channels 2 and 3 mix additively**. That table is a constant of the engine, not data. Channel 0 is the normal animation channel; the additive channels carry corrections — aim offsets, lean, recoil — layered over whatever channel 0 produced. A per-channel factor, settable at run time, scales a channel's whole contribution, which is how the game fades an aim layer in and out without touching the blends inside it.

**Blend.** One running instance of one motion on one part in one channel, with its own clock, its own weight, and its own accrue/falloff envelope. See [`animation_blend.h`](animation_blend.h.md) for the envelope; this interface only starts, stops and enumerates them.

## `IKinematicsAnimated` — what an implementor must provide

### Starting a cycle

```text
FUNCTION play_cycle(name : text, mix_in : bool, on_end : optional<fn>, payload : any, channel : int) -> optional<Blend ref>
FUNCTION play_cycle(motion : MotionID, mix_in, on_end, payload, channel) -> optional<Blend ref>
FUNCTION play_cycle(part : int, motion : MotionID, mix_in, on_end, payload, channel) -> optional<Blend ref>
```

**Contract** — starts a motion and returns a handle to the running blend, or nothing if the motion does not exist or the part is unnamed. The by-name form fails loudly: a missing cycle name is a content error and the engine says so. `mix_in` chooses whether the blends already on that part fade out over the new motion's falloff time or are cut dead. The callback fires once, when the motion reaches its end. The returned handle stays valid only while the blend is running — see the lifetime rule below.

The three forms differ only in where the body part comes from: not given at all (the motion's authored definition supplies it), or given explicitly by the caller (overriding the authored part, which is how the same animation is made to play on the torso for one creature and the whole body for another).

```text
FUNCTION play_cycle_explicit(part, motion, mixing, accrue, falloff, speed, no_loop,
                             on_end, payload, channel) -> optional<Blend ref>
  IF motion is invalid            -> RETURN none
  IF part == every_part           -> start on each named part in turn; RETURN none
  IF part >= PARTS or part unnamed-> RETURN none

  # Only channel 0 evicts what is already playing. The additive channels
  # layer without disturbing the base motion, which is the whole point of them.
  IF channel == 0
    IF mixing THEN fade_out_cycles(part, falloff, channel 0)
    ELSE           close_cycles(part, channel 0)

  blend = take_free_slot_from_pool()     # FAIL WITH "too many blended motions" if none
  initialize blend from (part, channel, motion, accrue, falloff, speed, no_loop, callback)
  FOR EACH bone IN part.bones
    register blend on bone
  append blend to cycles[part]
  RETURN blend
```

**Notes** — The envelope parameters are not passed by the caller in the common case; they come quantized from the motion's authored definition and are dequantized on read. The quantization is `value / 655.35` — that is, a 16-bit field spanning 0..100 — and accrue and falloff are then scaled by a fixed **1.5** before use. That constant has no discoverable derivation; it reads as a global "make blends 50% snappier than authored" correction that was applied once and frozen.

`play_cycle_explicit` returning nothing for the all-parts case is a real asymmetry: a caller that starts a motion on every part gets no handle to any of them and must stop them by part.

### Stopping and fading

```text
FUNCTION close_cycle(part : int, channel_mask : int)
```

**Contract** — ends every cycle on that part whose channel is in the mask, immediately and without a callback. The default mask is channel 0 alone, so an unqualified stop leaves the additive layers running — which is usually what the caller wants and occasionally a surprise.

```text
FUNCTION set_channel_factor(channel : int, factor : real)
```

**Contract** — scales a channel's contribution to every bone, from the next solve onward. Reset to one for every channel when the model is spawned or duplicated.

### Additive one-shots

```text
FUNCTION effect_id(name : text) -> MotionID          # fatal if absent
FUNCTION effect_id_safe(name : text) -> MotionID     # invalid handle if absent
FUNCTION play_effect(name_or_id, power_scale : real) -> optional<Blend ref>
FUNCTION play_effect_safe(name : text, power_scale) -> optional<Blend ref>
```

**Contract** — starts a bone-local additive motion: a flinch, a weapon's recoil kick, a vehicle's shudder. It attaches to a single bone (the motion's authored bone, or the skeleton root if it names none) and propagates down the hierarchy from there, rather than to a body part. `power_scale` multiplies the authored power, so the same flinch serves a graze and a serious hit. **Refuses silently when the effect list is already at `MAX_BLENDS`** — unlike cycles, running out of effect slots drops the effect instead of failing.

An effect's envelope has no steady state: it accrues to full weight, then immediately falls off, then frees itself. That is what makes it a one-shot rather than a pose.

**Notes** — The `_safe` variants exist because some effects are optional content. The pattern — a loud variant for names the code guarantees and a quiet one for names the data may omit — recurs across this interface and is worth preserving as two functions rather than one with a flag.

### Motion lookup

```text
FUNCTION cycle_id(name : text) -> MotionID           # fatal if absent
FUNCTION cycle_id_safe(name : text) -> MotionID      # invalid handle if absent
FUNCTION motion_slot_count() -> int
FUNCTION motion_slot(index) -> MotionBank
FUNCTION partitions() -> Partition
FUNCTION animation_length(motion : MotionID) -> real   # seconds
FUNCTION motion_def(motion) -> MotionDef
FUNCTION root_motion(motion) -> Motion                  # the root bone's key stream
FUNCTION motion(motion, bone_id) -> Motion              # one bone's key stream
FUNCTION part_id(name : text) -> int
```

**Contract** — lookup searches the motion banks **from the last loaded backwards**, so a bank loaded later shadows an identically named motion in an earlier one. This is the override mechanism: a creature loads the shared animation bank, then its own, and its own wins. A rebuild that searches forwards breaks every model that relies on it.

A `MotionID` is a (bank, index) pair, not a name — see [`animation_motion.h`](animation_motion.h.md). Resolving a name once and replaying the handle is the expected usage; the AI layer resolves thousands of names at load and never looks one up again.

### Advancing time

```text
FUNCTION update_tracks()
  IF now_ms == last_track_ms
    RETURN
  elapsed_ms = clamp(now_ms - last_track_ms, 0, 66)    # see note
  IF track_driver EXISTS
    # The owner takes over pacing entirely — this is how an entity animates
    # on its own simulation clock rather than the render clock.
    IF track_driver(real_elapsed_seconds, self)
      last_track_ms = now_ms
    RETURN
  last_track_ms = now_ms
  advance_tracks(elapsed_ms / 1000, force = false, keep_finished = false)
```

```text
FUNCTION advance_tracks(dt : real, force : bool, keep_finished : bool)
  FOR EACH part IN named parts
    FOR EACH blend IN cycles[part]
      IF NOT force AND blend.frame_stamp == current_frame
        CONTINUE                       # already advanced this frame by another caller
      blend.frame_stamp = current_frame
      IF blend.advance(dt) AND NOT keep_finished
        notify blend_destroy_listener
        unregister blend from its bones, free its pool slot, remove from cycles[part]
  advance_effect_tracks(dt)
```

**Contract** — advances every running blend's clock and envelope, retires the ones that finished, and fires their end callbacks. Called from the skeleton solve, so it runs at most once per instant per model whatever else happens.

**Invariants** — a blend removed here has already had its end callback fired by its own envelope update; the removal is what frees the pool slot and invalidates every handle to it.

**Notes** — Two clamps in eleven lines and both matter.

`elapsed_ms` is capped at **66 ms**. After a load screen, an alt-tab or a long frame, the real elapsed time can be seconds; letting that through would skip animations past their ends, fire their callbacks in a burst and teleport root motion. Capping means animation time runs slow rather than jumping — the same decision the simulation loop makes with its fixed-timestep accumulator, for the same reason.

The frame stamp is a second, finer guard: within one frame the same skeleton can be solved by the renderer and by a forced query, and without the stamp its blends would advance twice. `force` is the caller deliberately overriding that — used when replaying a sequence deterministically rather than in real time.

`keep_finished` suppresses retirement so a caller can inspect a finished blend before it is reclaimed. Only the deterministic-replay path uses it.

### Driver hooks

```text
FUNCTION set_track_driver(driver)         # driver(dt_seconds, skeleton) -> consumed : bool
FUNCTION track_driver() -> optional<driver>
FUNCTION set_blend_destroy_listener(listener)
FUNCTION blend_destroy_listener() -> optional<listener>
FUNCTION iterate_blends(visitor)          # every running blend, cycles and effects alike
FUNCTION part_blend_count(part) -> int
FUNCTION part_blend(part, n) -> Blend ref
```

**Contract** — the track driver replaces the default pacing wholesale; returning *not consumed* leaves the last-update stamp untouched so the skipped time is offered again next call. The blend-destroy listener is how an owner that cached a blend handle learns the handle just died: **the only safe lifetime rule for a blend handle is "valid until the destroy listener names it"**, and the engine's own animation manager is built on exactly that.

**Notes** — This is the interface's answer to a genuine ownership problem: blends live in a pool the skeleton owns, but the game holds references to them across frames to ask "is the reload still playing". Rather than reference counting, the engine notifies on death. A rebuild with a generational handle solves the same problem without the callback; what must survive is that **a blend handle is not an owning reference and must not be dereferenced after its motion ends**.

### Procedural animation

```text
FUNCTION gather_keys(bone : BoneData, channel_mask, out keys : KeyTable)
FUNCTION build_bone_matrix(out instance : BoneInstance, parent : Matrix, keys : KeyTable)
```

**Contract** — the two halves of a single bone's pose computation, exposed separately so a caller can insert its own work between them. `gather_keys` samples and dequantizes every blend covering the bone into a per-channel table; `build_bone_matrix` mixes that table down to one rotation and translation and composes it onto the parent.

```text
RECORD KeyTable
  keys   : Key[CHANNELS][MAX_BLENDS]    # sampled pose per contributing blend
  blends : Blend ref[CHANNELS][MAX_BLENDS]
  counts : int[CHANNELS]                # how many of each row are populated
```

```text
FUNCTION gather_keys(bone, channel_mask, out keys)
  FOR EACH blend covering bone
    IF blend.channel NOT IN channel_mask
      CONTINUE
    sample blend's key stream for this bone at blend.time -> keys.keys[channel][n]
    keys.blends[channel][n] = blend
    also sample the stream's FIRST key -> base[channel][n]
    n = n + 1
  # An additive channel contributes a *difference* from the motion's first frame,
  # not an absolute pose. Subtracting the base here is what makes layering work.
  FOR EACH channel whose external rule is additive
    keys.keys[channel] = keys.keys[channel] - base[channel]
```

```text
FUNCTION build_bone_matrix(out instance, parent, keys)
  FOR EACH channel with at least one key (channel 0 always participates)
    mixed[channel] = mix keys within the channel by each blend's weight, by the channel's internal rule
  result = mix the channel results by the channel table's external rules, scaled by channel factors
  instance.transform = parent . transform_from(result.rotation, result.translation)
```

**Notes** — Subtracting the motion's first key from an additive channel's samples is the single most important line in the animation system and appears nowhere in any comment. Without it, an aim-offset animation would *replace* the walk rather than bend it. A rebuild that authors additive motions as deltas at export time does not need it; one that reads the shipped banks does.

The fixed channel table — interpolate, interpolate, add, add — is a constant of the engine and shipped content depends on it. Channels 2 and 3 are the layer channels.

### Bone offsets and downcasts

```text
FUNCTION add_bone_offset(offset)          # forwards to the rigid skeleton
FUNCTION clear_bone_offset(bone_id)
FUNCTION on_calculate_bones()             # hook the solve calls: advances tracks
FUNCTION as_visual() -> RenderVisual
FUNCTION as_skeleton() -> Skeleton
```

**Contract** — `on_calculate_bones` is the hook by which the rigid skeleton's solve pulls animation forward: the solve calls it before deciding whether it may skip, so tracks advance at the real frame rate even when the pose itself is computed at the coarse rate. That ordering is load-bearing — invert it and animations stutter at ten frames a second.

### Debug surface

```text
FUNCTION motion_def_name(motion) -> (bank_name : text, motion_name : text)
FUNCTION dump_blends()
```

Debug builds only. Reverse-mapping a motion handle to its authored name is impossible without them, which makes them the first thing to restore when animation goes wrong.

## `IterateBlendsCallback`, `IUpdateTracksCallback`

**Contract** — one-method visitors, called synchronously, that must not start or stop blends while they run: the iteration is over live containers that a start or stop would reorder. A rebuild expresses both as ordinary function values.

## `SKeyTable`

**Contract** — the scratch buffer `gather_keys` fills and `build_bone_matrix` reads. It is stack-allocated per bone per solve and is sized for the worst case — every channel fully populated — which makes it large; the counts array is the only part that is initialized. Its existence is a decision: **pose mixing is a two-phase gather-then-combine, not a running accumulation**, because the additive channels need the whole channel's contents before they can be combined.
