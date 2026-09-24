# src/xrGame/ParticlesPlayer.cpp

> Plays particle effects on an animated object's bones: effects are attached to authored skeleton points, follow the pose every frame, age out, and die with their carrier.

**Needs** — [`ParticlesPlayer.h`](ParticlesPlayer.h.md) · [`ParticlesObject.h`](ParticlesObject.h.md) · [`xrEngine/xr_object.h`](../xrEngine/xr_object.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: skeleton transforms and per-frame re-placement; no explicit layout or device contact

## Purpose

Every wound that smokes, every muzzle that flashes, every engine that steams does it through
this mix-in. It answers three questions the effect object itself cannot: *where on this
model may an effect hang*, *how does it follow the animation*, and *who is allowed to stop
it*.

It is a mix-in rather than a component because the carrier must be a game object with a
skinned visual, and the effect placement reads the carrier's bone transforms every frame —
the coupling is total, so the split would buy nothing.

## State

```text
RECORD ParticlesInfo                    # one playing effect
  effect     : optional<ParticlesObject>   # none marks a slot ready to be reaped
  angles     : (heading, pitch, bank)      # orientation, stored as angles not as a matrix
  sender_id  : int (16-bit)                # entity identifier of whoever started it
  life_time  : int (ms)                    # sentinel "endless" = the all-ones value

RECORD BoneInfo                         # one authored attachment point
  index      : int (16-bit)                # bone index in the carrier's skeleton
  offset     : vector                      # placement in that bone's local frame
  particles  : list<ParticlesInfo>

RECORD ParticlesPlayer
  bones      : list<BoneInfo>
  bone_mask  : int (64-bit, bit per bone)  # invariant: bit i set exactly when some BoneInfo has index i
  carrier    : optional<GameObject>        # none exactly while the carrier is not spawned
  parent_vel : vector                      # velocity newly emitted particles inherit
  any_active : bool                        # cached: any live effect on any bone
```

Three invariants:

- **`bone_mask` mirrors `bones`.** The mask exists so that resolving an arbitrary bone to
  the nearest attachment point is a walk up the parent chain with a bit test per step,
  rather than a list scan per step. It is rebuilt whenever the bone list is.
- **The bone list is never empty.** If the model authors no attachment points, the list is
  seeded with the skeleton root at zero offset, so every start call finds somewhere to hang.
- **`carrier` is set exactly between spawn and destroy.** Both ends assert it, and the
  destructor asserts it is clear — an effect carrier that is torn down while spawned would
  leave effect objects with a dangling parent.

The 64-bit mask caps attachment bones at bone index 63, not at 64 attachment points. A model
whose particle bones sit above index 63 silently loses them. Nothing in the source
acknowledges this; a rebuild should use a set keyed by bone index instead, and lose nothing.

## `LoadParticles`

**Contract** — reads the attachment points from the model's own embedded configuration
section, once, when the carrier's visual is known. Each entry names a bone and gives a
three-component local offset. Refuses to load if a named bone is not in the skeleton — this
is authoring data, and a typo that silently drops an effect is worse than a stop.

```text
FUNCTION load_particles(skeleton)
  bones.clear()
  section = skeleton.embedded_config["particle_bones"]
  IF section exists
    bone_mask = 0
    FOR EACH (bone_name, offset_text) IN section
      index = skeleton.bone_index(bone_name)
      IF index is none: FAIL WITH "particles bone not found: " + bone_name
      bones.append(BoneInfo(index, parse_vector(offset_text)))
      bone_mask = bone_mask OR bit(index)
  IF bones is empty
    bone_mask = bit(0)                 # see Notes
    bones.append(BoneInfo(skeleton.root_bone, zero))
```

**Notes** — the fallback sets the mask bit for bone *zero* while the seeded attachment point
uses the skeleton's declared root bone. These are the same bone on every shipped model, and
the code is only correct because of that. A rebuild should set the bit for whichever bone it
actually seeded.

The attachment points live in the *model*'s configuration, not in the entity's section. That
is the right owner: where a wound may smoke is a property of the mesh, and the same mesh is
shared by many entity sections.

## `StartParticles`

**Contract** — begin playing a named effect. Four spellings collapse to two decisions:
*which bones* (one nominated bone, resolved to the nearest attachment point above it in the
skeleton; or every attachment point at once) and *what orientation* (a direction, from which
an orthonormal basis is generated with the direction as the forward axis; or a caller-built
transform whose translation must be zero, since the translation comes from the bone).
Starting an effect that is already playing on that bone re-targets and re-tags the existing
one rather than stacking a second copy. Allocates on first use of a name per bone.

**Invariants** — the supplied transform carries no translation; the bone supplies it.
Afterwards `any_active` is true.

```text
FUNCTION start_particles(name, bone, orientation, sender_id, life_time, auto_stop)
  target = nearest_attachment_bone(bone)     # walk parents until bone_mask has the bit
  IF target is none: RETURN                  # no attachment above it: silently nothing
  info = target.find_or_create(name)
  info.sender_id = sender_id
  info.life_time = auto_stop ? life_time : endless
  info.angles    = orientation.as_angles()   # see Notes
  placement      = matrix from info.angles
  placement.translation = world_position_of(target.index, target.offset)
  info.effect.update_parent(placement, zero_velocity)
  IF NOT info.effect.is_playing(): info.effect.play(not_hud)
  any_active = true
```

**Notes** — the orientation is decomposed into three angles at start and recomposed from
them every frame. That is not a rounding accident; it is what makes the effect keep its
authored *world* orientation while the bone moves under it. A muzzle flash re-derived from
the bone's own rotation would spin with the weapon's recoil animation. A rebuild may store
a rotation any way it likes, but it must store the *world* rotation fixed at start and only
take the *position* from the bone each frame.

The all-bones spelling differs in one more way: it uses the caller's transform rotation
directly rather than the angle round-trip. The two paths therefore disagree slightly about
orientation, which is invisible because the all-bones spelling is used for whole-body
effects that are radially symmetric.

An effect started on a bone with no attachment point above it is dropped without complaint,
while an attachment bone missing from the skeleton is a hard failure at load. That asymmetry
is right: the first is a runtime hit landing on an arbitrary bone, the second is bad data.

## `StopParticles` / `AutoStopParticles`

**Contract** — two ways to select what to stop: by the identifier of whoever started it, or
by effect name; and two scopes: one nominated attachment bone, or all of them. A destroy
flag chooses between letting live particles finish (the effect object's deferred stop) and
killing them immediately. Both forms end by running the per-frame pass once, so that
finished effects are reaped without waiting for the next frame.

`AutoStopParticles` does not stop anything — it gives a matching running effect a remaining
lifetime, converting an endless effect into an expiring one. This is how an effect started
"until further notice" is later told it has two seconds left.

**Notes** — stopping by starter identifier is the load-bearing one. A carrier may have
effects from several sources on the same bone (a wound and a weapon flash on the same
shoulder), and each source must be able to retract only its own without knowing the names
of anyone else's.

## `UpdateParticles`

**Contract** — the per-frame pass, called by the carrier. Re-places every live effect from
its bone's current pose, ages it, stops it when its time is up, destroys it when it has
finished playing, and compacts the lists. Returns immediately when nothing is active, which
is the common case and the reason `any_active` is cached at all.

```text
FUNCTION update_particles()
  IF NOT any_active: RETURN
  any_active = false                         # re-derived by this pass
  FOR EACH bone IN bones
    FOR EACH info IN bone.particles
      IF info.effect is none: CONTINUE
      placement = matrix from info.angles
      placement.translation = world_position_of(bone.index, bone.offset)
      info.effect.update_parent(placement, parent_vel)
      IF info.life_time is not endless
        IF info.life_time > frame_delta
          info.life_time = info.life_time - frame_delta
        ELSE
          info.effect.stop(deferred)         # let the tail finish
          info.life_time = endless           # stop re-triggering; the reap below ends it
      IF NOT info.effect.is_playing()
        destroy(info.effect)                 # clears info.effect to none
      ELSE
        any_active = true
    bone.particles.remove entries whose effect is none
```

**Invariants** — after the pass, `any_active` is true exactly when some effect is still
playing, and no list holds a reaped slot.

**Notes** — the ordering is load-bearing in two places. Expiry sets the remaining lifetime
back to the endless sentinel *after* asking for a deferred stop, so the timer does not fire
again on every subsequent frame while the tail plays out; the effect's own not-playing
report is what finally reaps it. And placement happens before ageing, so an effect is drawn
in the right place on the frame it expires rather than one frame stale.

The effects are re-placed with the carrier's velocity rather than zero, so newly emitted
particles inherit the carrier's motion and trail behind it. That velocity is pushed in from
outside by the carrier; nothing here derives it.

## `net_SpawnParticles` / `net_DestroyParticles`

**Contract** — the lifecycle bracket. Spawn resolves and stores the carrier handle — the
mix-in must discover the game object it is part of, since it is not itself one. Destroy
tears down every effect on every bone immediately (not deferred) and clears the carrier.

**Invariants** — destroy leaves the bone list intact but every effect list empty, so a
carrier that respawns re-uses the loaded attachment points without re-reading the model.

**Notes** — destruction is immediate, not deferred. When a carrier disappears its smoke goes
with it, because a deferred tail would keep re-placing itself from a skeleton that no longer
exists. That is the whole reason this bracket exists rather than letting effects expire on
their own.

## `get_nearest_bone_info` / `GetNearestBone`

**Contract** — resolve an arbitrary bone to the nearest attachment point at or above it in
the skeleton, by walking the parent chain and testing the bone mask at each step. Yields
nothing if the chain reaches the root without a hit. The two spellings differ only in
returning the attachment record versus the bone index.

**Notes** — this is what makes hit effects work without authoring an attachment point on
every bone. A bullet lands on a finger; the smoke plays from the nearest authored point,
which is the hand or the forearm. The walk is upward only, so an attachment point on a child
bone is never found from its parent — placement degrades toward the torso, never outward,
which is the visually forgiving direction.

## `GetBonePos` / `MakeXFORM`

**Contract** — static, usable without a carrier instance. `GetBonePos` takes a local offset
in a bone's frame through the bone's current pose and then the object's world transform,
yielding a world point. `MakeXFORM` additionally builds an orthonormal basis from a
direction and drops that world point into its translation, producing a complete world
placement in one call.

**Notes** — the two-step transform (bone, then object) is the invariant worth keeping: bone
transforms are in the model's space, not the world's, and a rebuild that forgets the second
step gets effects that sit at the world origin and drift correctly with the animation, which
is a confusing symptom to diagnose.

## `GetRandomBone`

**Contract** — an arbitrary attachment point, uniformly chosen; nothing if there are none.
Used where an effect should appear *somewhere* on a creature without the caller knowing its
anatomy.
