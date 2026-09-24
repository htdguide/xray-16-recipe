# src/Layers/xrRender/SkeletonAnimated.cpp

> The animation player: motion banks loaded from the shipped animation files, a fixed pool of blends distributed across body parts and channels, the per-bone blend lists, and the sample-dequantize-mix step that turns them into one bone pose.

**Needs** — [`SkeletonAnimated.h`](SkeletonAnimated.h.md) · [`SkeletonCustom.h`](SkeletonCustom.h.md) · [`SkeletonX.h`](SkeletonX.h.md) · [`Animation.h`](Animation.h.md) · [`AnimationKeyCalculate.h`](AnimationKeyCalculate.h.md) · [`KinematicAnimatedDefs.h`](KinematicAnimatedDefs.h.md) · [`xrCore/Animation/SkeletonMotions.hpp`](../../xrCore/Animation/SkeletonMotions.hpp.md) · [`xrCore/FMesh.hpp`](../../xrCore/FMesh.hpp.md) · [`Include/xrRender/KinematicsAnimated.h`](../../Include/xrRender/KinematicsAnimated.h.md)
**Used by** — reached through its declarations in [`SkeletonAnimated.h`](SkeletonAnimated.h.md); callers name that, not this file.
**Tier floor** — T1: animation keys are read as quantized byte images out of a mapped file and dequantized in the inner loop, the blend pool is a fixed preallocated array, and the per-bone blend lists are fixed-capacity inline arrays sized so that a bone's whole list fits in cache.

## Purpose

This file implements the animated skeleton: the rigid skeleton from [`SkeletonCustom.cpp`](SkeletonCustom.cpp.md) plus everything needed to drive it from authored motion data. It owns four things that exist nowhere else:

- **the motion banks** — which animation files this model loaded, in what order, and how a name resolves to a motion;
- **the blend pool** — a fixed set of running-animation slots, sized so a model can never need more;
- **the per-bone blend lists** — the reverse index that makes posing one bone a local operation;
- **the pose step** — sample every blend covering a bone, dequantize, subtract the additive base, mix within each channel, mix across channels, compose onto the parent.

The interface it satisfies is described in [`Include/xrRender/KinematicsAnimated.h`](../../Include/xrRender/KinematicsAnimated.h.md); this page describes how, and what the numbers are.

## State

```text
RECORD AnimatedSkeleton                     # extends the rigid skeleton
  motion_slots    : list<MotionSlot>        # SHARED with every clone; at most 48
  partition       : Partition               # SHARED; 4 named body parts
  blend_instances : list<BlendInstance>     # PER-CLONE, parallel to bones
  blend_pool      : list<Blend> of 256      # PER-CLONE, fixed, never grows
  blend_cycles    : list<Blend ref>[4]      # PER-CLONE; running cycles, per body part
  blend_fx        : list<Blend ref>         # PER-CLONE; running additive one-shots
  channels        : ChannelFactors          # PER-CLONE; one scalar per channel
  track_time_ms   : int                     # wall clock of the last track advance
  track_driver    : optional<driver>
  blend_destroy_listener : optional<listener>

RECORD MotionSlot                           # one loaded animation file
  motions      : MotionBank                 # name tables and motion definitions
  bone_motions : list<list<Motion>>         # indexed by THIS model's bone id, then by motion index

RECORD BlendInstance                        # per bone: which blends cover it
  blends : list<Blend ref>, at most 16
```

**The fixed numbers, all frozen against shipped data:**

```text
body parts per model        : 4     # the partition table in an animation bank has four entries
channels                    : 4     # a motion definition names its channel in a two-bit field
blends per bone             : 16
blend pool size             : 256   # = 16 blends x 4 parts x 4 channels: the worst case, preallocated
animation files per model   : 48
track advance cap           : 66 ms
```

**Invariants**

- **The blend pool never grows, and exhausting it is fatal.** The pool is sized for the arithmetic worst case, so exhaustion means a leak — blends started and never retired — not a busy model. The engine would rather stop than silently play the wrong animation.
- A pool slot is free exactly when its curvature state is the free marker. Nothing else distinguishes a free slot; there is no separate free list.
- A part with no name is not a part. Playback on an unnamed part index is silently a no-op, which is how models with fewer than four body parts work.
- The motion slots and the partition are **shared with every clone**; the blend pool, the per-bone lists and the channel factors are **private to each**.
- A blend's `bone_or_part` field means a *part index* for a cycle and a *bone index* for an additive one-shot. The two populations never mix, and the field is read according to which list the blend is in.
- A blend reference is valid only while the blend is running. The only safe lifetime rule for a holder is "valid until the destroy listener names it".

## The four concepts, and what makes each one necessary

**Body part.** The bones are partitioned into up to four named groups by data authored alongside the animations. A cycle plays *on a part*, so the legs can walk while the torso reloads.

**Channel.** Orthogonal to parts. Each blend declares one of four channels, and the channels combine by a **fixed rule table that is a constant of the engine, not data**: two channels interpolate, two add. The additive channels carry corrections — aim offset, lean, recoil — layered over whatever the interpolating channels produced. A per-channel scalar, settable at run time, scales a whole channel's contribution.

**Cycle versus effect.** A *cycle* is a motion on a body part with a steady state: it accrues to weight, holds, and either loops or stops. An *effect* is an additive one-shot attached to **a single bone and propagated down its subtree**: it accrues to full power and immediately falls off to nothing, then frees itself. A flinch, a weapon's kick, a vehicle's shudder. The two are stored in separate lists and advanced by separate loops because their envelopes have different shapes.

**The per-bone blend list.** Posing a bone must not search every running blend. When a blend starts, it registers itself on every bone it covers; when it ends, it unregisters. The bone then carries the (short) list of blends that affect it, and posing it is local.

## `Load` — finding the animation banks

**Contract** — after the rigid skeleton has loaded, attach this model's motion banks. Three mutually exclusive sources, checked in order. Fatal if a referenced animation file cannot be found, or if no bank loads at all. Blocks on file I/O and on directory listing.

```text
FUNCTION load(name, reader)
  load the rigid skeleton first

  IF chunk 19 (motion references) EXISTS
    one string containing a LIST of bank names
    FAIL WITH fatal IF the list holds 48 or more entries
    FOR EACH entry: load_bank(entry + ".omf")        # or expand a wildcard, below

  ELSE IF chunk 24 (motion references, second form) EXISTS
    a count, then that many separate strings
    FOR EACH: load_bank(entry + ".omf")              # or expand a wildcard

  ELSE
    the motions are embedded in the model file itself; make one bank from it

  FAIL WITH fatal IF no bank loaded

  partition = the FIRST bank's partition, resolved against this skeleton's bone names

  FOR EACH bank
    FOR EACH bone IN index order
      bank.bone_motions[bone] = the bank's key streams for that bone's NAME
```

```text
FUNCTION load_bank(path)
  file = first that exists of $level$/path, $game_meshes$/path
  FAIL WITH fatal IF neither exists
  IF the shared bank container does not already hold this path
    read and register it
  register this model's use of it
```

**Notes**

- **Two chunk forms for the same information**, and both must be read: the older one packs the bank list into a single delimited string, the newer one writes a count and separate strings. The shipped data contains both, because models were exported across several years.
- **A bank name may be a wildcard ending in `\*.omf`**, which is expanded by listing both search roots. This is how a creature says "load every animation file in my folder", and it means the set of banks a model loads is a property of the *installed files*, not only of the model. A rebuild must implement the listing, or those models animate with an incomplete motion set.
- **Bone motions are bound by name, not by index.** A shared animation bank is authored against a canonical skeleton; each model that uses it maps its own bones onto it by name. That is what lets one bank drive several models, and it is why a renamed bone silently loses its animation instead of failing.
- **The partition comes from the first bank only.** A model whose banks disagree about body parts uses the first one's answer. Nothing checks.
- The bank cache is consulted before reading, so the file cost is paid once per bank per session however many models reference it. The double registration — once with the file, once without — is the cache's own two-step admission protocol and is incidental.

## Motion lookup

**Contract** — resolve a motion name to a (bank, index) handle, searching **the banks from the last loaded backwards**. Separate tables for cycles and effects, plus a combined table. A loud variant fails on a missing name; a quiet one returns an invalid handle.

**Notes** — The backwards search is the **override mechanism** and it is load-bearing: a creature loads the shared bank first and its own bank second, and its own wins on any name collision. A rebuild that searches forwards breaks every model that relies on it, which is most of them.

Resolving a name is a map lookup per bank; the expected usage is to resolve once at load and replay the handle forever, which is what the AI layer does.

## Starting a cycle

**Contract** — start a motion on a body part in a channel, returning a handle to the running blend or nothing. Retires or fades whatever was playing, takes a pool slot, registers the blend on every bone of the part.

```text
FUNCTION play_cycle(part, motion, mixing, accrue, falloff, speed, no_loop, callback, payload, channel)
  IF motion is invalid: RETURN none
  IF part is the sentinel                     # "every part"
    start on each of the four parts in turn; RETURN none
  IF part is out of range OR unnamed: RETURN none

  # Only channel 0 evicts. The additive channels layer without disturbing the
  # base motion, which is the entire reason they exist.
  IF channel = 0
    IF mixing THEN fade_cycles(part, falloff, channel 0)
    ELSE           close_cycles(part, channel 0)

  blend = take_free_pool_slot()               # FAIL WITH fatal if the pool is exhausted
  setup_cycle(blend, part, channel, motion, mixing, accrue, speed, no_loop, callback, payload)
  FOR EACH bone IN part.bones
    register blend on that bone               # NOT recursive: the part already lists every bone
  append blend to cycles[part]
  RETURN blend
```

```text
FUNCTION setup_cycle(blend, part, channel, motion, mixing, accrue, speed, no_loop, ...)
  blend.state  = accruing
  blend.weight = IF mixing THEN almost-zero ELSE 1     # not mixing means snap to full weight
  blend.accrue = accrue
  blend.falloff = 0                           # a blend's own falloff is set only when it is faded out
  blend.power  = 1
  blend.speed  = speed
  blend.time_current = 0
  blend.time_total   = length of this motion's ROOT bone key stream
  blend.bone_or_part = part
  blend.stop_at_end  = no_loop
  blend.playing = true ; blend.stop_at_end_callback = true
  blend.channel = channel
  blend.fall_at_end = no_loop AND channel > 1
```

**Notes**

- **The envelope values are not usually supplied by the caller**; the convenience forms read them from the motion's own authored definition — accrue, falloff, speed, and whether it loops — and also take the body part from there unless the caller overrode it. That is how the same call site drives motions with wildly different blend timings without knowing anything about them.
- `blend.falloff` is deliberately zeroed at start: the parameter named falloff at this level is the falloff applied to the *previous* cycles being faded out, not to the new one. The new blend acquires a falloff only when something later fades it. Naming one parameter for two roles is the source's own confusion; a rebuild should split them.
- **`fall_at_end` is set only for a non-looping motion on an additive channel.** A finished blend on channel 0 or 1 holds its last pose (the creature stays where the animation left it); a finished blend on an additive channel must *fall off*, because an additive layer stuck at full weight would bend the pose forever. The number 1 in that comparison is the channel index boundary of the fixed interpolate/add rule table, restated in a second place — a rebuild should derive it from the table instead.
- **A start on the sentinel "all parts" returns nothing**, so a caller who starts a motion on the whole body gets no handle to any of the four blends and must stop them by part.

## Fading and closing

```text
FUNCTION fade_cycles(part, falloff, channel_mask)
  FOR EACH blend ON part whose channel is in the mask
    blend.state   = falling off
    blend.falloff = falloff
    IF blend.stop_at_end
      blend.stop_at_end_callback = false      # a faded-out blend must NOT report that it finished
```

```text
FUNCTION close_cycles(part, channel_mask)
  FOR EACH blend ON part whose channel is in the mask
    free its pool slot
    unregister it from every bone of its part
    remove it from the part's list
```

**Notes** — Suppressing the end callback when a blend is faded out is the distinction between *this animation completed* and *this animation was replaced*. Game code listens for completion to advance a state machine; firing it on a replacement would advance the machine on an animation that never finished. This single flag is what keeps the two events apart.

Closing is immediate and silent — no callback, no fade — and is what a caller asks for when the character's state changed so abruptly that blending would look wrong.

## Starting an effect

```text
FUNCTION play_effect(bone, motion, accrue, falloff, speed, power)
  IF motion is invalid: RETURN none
  IF the effect list already holds 16: RETURN none     # dropped, not fatal
  IF bone is the sentinel: bone = root
  blend = take_free_pool_slot()
  setup_effect(blend, motion, accrue, falloff, power, speed, bone)
  register blend on that bone AND ON EVERY DESCENDANT   # recursive, unlike a cycle
  append to the effect list
```

```text
FUNCTION setup_effect(blend, motion, accrue, falloff, power, speed, bone)
  blend.state = accruing ; blend.weight = almost-zero
  blend.accrue = accrue ; blend.falloff = falloff ; blend.power = power
  blend.speed = speed ; blend.time_current = 0
  blend.time_total = length of this motion's key stream FOR THAT BONE
  blend.bone_or_part = bone
  blend.channel = 0 ; blend.stop_at_end = false ; blend.fall_at_end = false
  blend.callback = none
```

**Notes**

- **Effects register recursively down the hierarchy; cycles do not.** A body part is an explicit bone list that already contains the whole group, so a cycle registers on exactly those bones. An effect names one bone and means "this bone and everything below it", so it walks. Both reach the same data structure by different routes, and confusing them registers a blend on the wrong set of bones.
- **Running out of effect slots drops the effect silently**, where running out of cycle slots is fatal. The difference is judgement, not accident: a lost flinch is invisible, a lost walk cycle is a frozen character. The limit compared against is the per-bone blend capacity, applied here to a global list — an arguably wrong comparison that happens to be a reasonable cap.
- An effect's declared channel is always 0, even though its *purpose* is additive layering. The layering comes from the blend's own envelope and from the caller's choice of motion, not from the channel table. This is a genuine inconsistency in the design.
- The effect's duration is read from **the attached bone's** key stream, not the root's, because an effect may be authored for a single limb and have no root-bone keys at all.

## The per-bone blend list

```text
FUNCTION register(bone, blend)
  IF the bone already holds 16 blends
    IF blend.fall_at_end
      RETURN                                  # drop the newcomer rather than evict
    evict the blend with the LOWEST current weight
  append blend
```

**Notes** — When a bone is oversubscribed, the blend contributing least to its pose is the one dropped, which is the least visible choice available. The exception is that a blend which will fade itself out is *never* allowed to evict an established one — a transient must not displace a steady state. Note that the evicted blend is removed only from *this* bone: it keeps running, keeps its pool slot, and still affects every other bone it covers. A bone can therefore be missing one contribution while its neighbour has it, which is a visible artefact in the rare case it happens and is accepted as cheaper than the alternative.

## Advancing time

```text
FUNCTION update_tracks()
  IF now_ms = last_track_ms: RETURN
  elapsed_ms = min(now_ms - last_track_ms, 66)
  IF track_driver EXISTS
    IF track_driver(real_elapsed_seconds, self)    # the driver gets the UNCLAMPED time
      last_track_ms = now_ms
    RETURN
  last_track_ms = now_ms
  advance(elapsed_ms / 1000, force = false, keep_finished = false)
```

```text
FUNCTION advance(dt, force, keep_finished)
  FOR EACH named part
    FOR EACH blend ON that part
      IF NOT force AND blend.frame_stamp = current_frame: CONTINUE
      blend.frame_stamp = current_frame
      IF blend.update(dt, blend.callback) AND NOT keep_finished
        retire it: notify the destroy listener, free the slot,
                   unregister from the part's bones, remove from the list
  advance_effects(dt)
```

```text
FUNCTION advance_effects(dt)
  FOR EACH blend IN the effect list
    IF NOT blend.stop_at_end_callback
      blend.playing = false ; CONTINUE          # frozen: held, not advanced, not retired
    advance its clock
    IF accruing
      weight = weight + dt * accrue * power * speed
      IF weight >= power: weight = power; switch to falling off
    ELSE IF falling off
      weight = weight - dt * falloff * power * speed
      IF weight <= 0
        free the slot, unregister from the bone AND ITS SUBTREE, remove from the list
```

**Notes**

- **The 66 ms cap** is the same decision the simulation loop makes with its fixed-timestep accumulator: after a load screen or a long frame the real elapsed time can be seconds, and letting that through would skip animations past their ends, fire a burst of callbacks and teleport root motion. Animation time runs *slow* rather than jumping. The number is roughly four frames at 60 Hz.
- **The frame stamp is a second, finer guard.** Within one frame the same skeleton can be solved by the renderer and by a forced query; without the stamp its blends would advance twice. `force` is a caller deliberately overriding that, used when replaying a sequence deterministically.
- **The track driver receives the unclamped elapsed time** and may decline the update by returning false, in which case the last-update stamp is left alone and the skipped time is offered again next call. This is how an entity animates on its own simulation clock instead of the render clock.
- **An effect's envelope has no steady state**: it accrues straight into falloff with no hold. The rates are scaled by both the blend's power and its speed, so a powerful, fast effect is also a *short* one — raising power raises the peak and shortens the envelope together, which is usually what an author wants from "hit harder" and is worth knowing before retuning it.
- A frozen effect (one whose completion reporting was suppressed) is left in the list forever, holding its pool slot and its registrations. Nothing in this file ever clears that state, so it is only safe because the only writer of it is the effect path itself.

## Taking a pool slot

```text
FUNCTION take_free_pool_slot() -> Blend ref
  update_tracks()                     # so a blend that ended this instant is available now
  FOR EACH slot IN the pool
    IF slot is free: RETURN it
  FAIL WITH fatal "too many blended motions"
```

**Notes** — Advancing tracks before searching is what makes the pool's arithmetic sizing sufficient: without it, a caller that starts a blend every frame could exhaust the pool with blends that have already finished but not yet been retired. It also means **starting an animation advances time**, which is a surprising side effect worth keeping visible in a rebuild.

## Posing a bone

Two halves, exposed separately so a caller can insert work between them.

### Gathering and dequantizing

```text
FUNCTION gather_keys(bone, channel_mask, out table)
  FOR EACH blend registered on this bone
    IF blend.channel not in channel_mask: CONTINUE
    stream = this blend's motion, for THIS bone
    n = table.count[blend.channel]
    table.keys[channel][n]   = sample stream at blend's current time, dequantized
    table.blends[channel][n] = blend
    base[channel][n]         = stream's FIRST key, dequantized the same way
      # rotation always present; translation present only if the stream has a
      # translation track, in one of two quantizations — otherwise the stream's
      # constant initial translation is used
    count[channel] = n + 1

  FOR EACH channel whose external rule is ADD
    subtract base from keys, entry by entry
```

**Notes** — **Subtracting the motion's own first key from an additive channel's samples is the single most important line in the animation system, and it carries no comment.** Without it, an aim-offset animation would *replace* the walk instead of bending it: what the layer must contribute is the *difference* from its own rest frame, not its absolute pose. A rebuild that authors additive motions as deltas at export time does not need it; a rebuild that reads the shipped banks does.

The translation track is optional per motion and, when present, comes in **two quantizations distinguished by a flag** — a compact one and a wider one. A motion with no translation track carries one constant translation instead. All three cases must be decoded, because the shipped banks contain all three; this is where most of a bank's size saving comes from, since most bones only rotate.

### Mixing

```text
FUNCTION build_bone_matrix(out instance, parent, table)
  FOR EACH channel
    IF channel is not 0 AND has no keys: SKIP        # channel 0 always participates
    mix the channel's keys among themselves by each blend's weight,
      using the channel's INTERNAL rule
  mix the channel results together by their EXTERNAL rules, scaled by the channel factors
  instance.transform = parent composed with a transform built from the mixed rotation and translation
```

**Notes** — Channel 0 participates even with no keys, so a bone covered by no blend at all still produces a defined pose rather than an uninitialized one. Every other channel is skipped when empty, which is the common case and is what keeps the cost proportional to what is actually playing.

Each channel has *two* rules: an internal one for combining blends within it, and an external one for combining its result with the other channels. Only the external rule is the interpolate/add distinction; the internal one is how several simultaneous blends on the same channel are weighted against each other. Keeping them separate is what allows two crossfading walk cycles on channel 0 to interpolate between themselves while the whole channel is then interpolated against the layers.

### The animated override

```text
FUNCTION build_bone_matrix(bone, instance, parent, channel_mask)     # overrides the rigid version
  table = gather_keys(bone, channel_mask)
  build_bone_matrix(instance, parent, table)
  apply_bone_offsets(bone, instance, parent, channel_mask)
```

The key table is a stack scratch buffer, sized for the worst case — every channel fully populated — and only its counts are initialized. Its existence encodes a real decision: **pose mixing is a two-phase gather-then-combine, not a running accumulation**, because the additive channels need a whole channel's contents before they can be combined.

## `OnCalculateBones`

**Contract** — the hook the rigid skeleton's solve calls before it decides whether it may skip: advance the tracks. One line, and it is the reason animation time runs at the frame rate while poses are computed at the coarse rate.

## Lifecycle

```text
FUNCTION copy(from)                # clone from the base
  copy the rigid skeleton
  SHARE: motion slots, partition
  reinitialize the blend pool, the part lists, the effect list, the channel factors

FUNCTION spawn()
  spawn the rigid skeleton
  reinitialize the blend pool and every per-bone blend list
  clear the track driver; reset every channel factor to one

FUNCTION blend_pool_startup()
  refill the pool with free slots, clear all four part lists and the effect list,
  reset the channel factors
```

**Invariants** — a clone starts with **no animations running**, whatever the base was doing. Nothing is inherited across the clone boundary except the data.

## Diagnostics

**Contract** — debug builds only: reverse-map a motion handle to its authored name and bank name (a linear scan of the bank's name table), and dump every pool slot with its timings, weights, state, part, channel, flags and callback.

**Notes** — Reverse-mapping a motion handle is impossible without the scan, which makes these the first thing to restore when animation goes wrong: every other diagnostic in the system reports opaque numeric handles.

## Could not recover

- Why the effect list's capacity is compared against the *per-bone* blend limit rather than a limit of its own. The number is defensible as a cap; the derivation is not.
- The per-channel *internal* mixing rules and the exact shape of a blend's weight update live in the animation support files, not here; this file only selects them.
- Whether an effect frozen by a suppressed completion report is ever cleaned up. Nothing in this file does it, and no other writer of that flag is visible from here.
