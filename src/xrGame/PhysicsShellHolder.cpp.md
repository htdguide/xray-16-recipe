# src/xrGame/PhysicsShellHolder.cpp

> The base of every game object that can own a rigid-body assembly, and the adapter through which the physics module asks the game layer questions it is not allowed to know the answers to.

**Needs** — [`PhysicsShellHolder.h`](PhysicsShellHolder.h.md) · [`GameObject.h`](GameObject.h.md) · [`ParticlesPlayer.h`](ParticlesPlayer.h.md) · [`CharacterPhysicsSupport.h`](CharacterPhysicsSupport.h.md) · [`PHMovementControl.h`](PHMovementControl.h.md) · [`ph_shell_interface.h`](ph_shell_interface.h.md) · [`Level.h`](Level.h.md) · [`CustomRocket.h`](CustomRocket.h.md) · [`Grenade.h`](Grenade.h.md) · [`xrPhysics/PhysicsShell.h`](../xrPhysics/PhysicsShell.h.md) · [`xrPhysics/IPhysicsShellHolder.h`](../xrPhysics/IPhysicsShellHolder.h.md) · [`xrPhysics/PHCommander.h`](../xrPhysics/PHCommander.h.md) · [`xrPhysics/IPHWorld.h`](../xrPhysics/IPHWorld.h.md) · [`xrPhysics/IActivationShape.h`](../xrPhysics/IActivationShape.h.md) · [`xrEngine/IObjectPhysicsCollision.h`](../xrEngine/IObjectPhysicsCollision.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: it owns a body assembly and quantizes poses into a wire format, but the byte layout is delegated

## Purpose

Two jobs, and they should probably be two types in a rebuild.

The first is **ownership**: this is where a game object gains a physics shell — the rigid
body or jointed assembly that the dynamics seam steps — together with the shell's sound
player, the routines that bring a shell into existence in each of the two ways an object
can acquire one, and the rules for tearing it down.

The second is **adaptation**: the physics module must be able to ask about the object it
is simulating — its transform, its name, its identity, whether it is the player, whether
it should collide with bullets, what its collision model is — without depending on the
game layer's type hierarchy. This file answers all of those by forwarding to the game
object underneath. Every one of those answers is a one-line delegation; taken together
they *are* the contract between the two modules, which is why they are listed rather than
elided.

Almost everything else in the class is a virtual returning nothing, overridden by
subclasses that actually have the thing being asked for (a destructible part, a collision
damage receiver, a character's movement control, an inverse-kinematics controller). The
base answering "no" to all of them means a caller never has to know what kind of object it
holds.

## State

```text
RECORD PhysicsShellHolder EXTENDS GameObject, ParticlesPlayer
  shell         : optional<PhysicsShell>   # the body assembly; absent until activated
  sound_player  : PhysicsSoundPlayer       # owned for the object's whole life, shell or not
  scheduled     : bool                     # whether this object wants scheduled updates
```

**Invariants**

- The shell is destroyed before the object is, and the teardown asserts it: an object
  reaching its destructor with a live shell means the dynamics world still holds a pointer
  into memory about to be freed. This is conformance's "a destroyed entity is
  unreferenced by the physics world before its memory is released".
- The shell cannot outlive the visual it was built from, because its bodies are indexed by
  the model's bones. Changing the visual to none destroys the shell (see `OnChangeVisual`).
- The sound player exists from construction, not from shell activation, because collision
  sounds are requested by the dynamics callback which may fire before the game layer has
  noticed the shell exists.

There is one further piece of state that is not a field, and it is the ugliest decision in
the file — described under `load`.

## `net_Spawn`

**Contract** — brings the object online. Spawns the attached particle emitters, clears the
parked enable-state (see `load`), marks the object as wanting scheduled updates, and runs
the game object's spawn, which is what reads the configuration section and, for subclasses
that build their shell at spawn, creates it.

If a fully active shell exists when the base spawn returns, its pose becomes the object's
authoritative transform (the solver, not the spawn record, is now the source of truth),
the parked enable state is applied — waking the body or putting it to sleep — and the
object's spawn-time configuration overrides are pushed into the shell.

```text
FUNCTION net_Spawn(server_record) -> bool
  particles.spawn()
  parked_enable_state <- undetermined
  scheduled <- true
  ok <- base.net_Spawn(server_record)        # reads the section, may build the shell
  IF shell EXISTS AND shell.is_fully_active() THEN
    transform   <- shell.global_transform()  # solver wins over the record
    shell.transform <- transform
    IF   parked_enable_state = enable  THEN shell.wake()
    ELSE IF parked_enable_state = disable THEN shell.sleep()
    # undetermined: leave the solver's own decision alone
    apply_spawn_configuration_to(shell)
    parked_enable_state <- undetermined
  RETURN ok
```

**Notes** — "undetermined" is a real third state, not a missing value: an object spawning
fresh from the level's spawn file has never been saved, so nobody knows whether it should
be awake, and forcing either answer is wrong. Only an object restored from a save carries
a decision.

## `net_Destroy`

**Contract** — takes the object offline. The order is the whole content of the routine and
a rebuild must reproduce it:

```text
FUNCTION net_Destroy()
  physics_command_queue.remove_all_calls_referencing(self)  # they hold a raw reference
  particles.destroy()
  IF character_support EXISTS THEN character_support.destroy_interactive_motion()
  base.net_Destroy()
  scheduled <- false
  deactivate_shell()          # withdraw bodies from the dynamics world
  destroy_shell()
```

**Invariants** — the deferred physics-script call queue is drained *first*, because those
queued calls name this object and would run against freed memory. The interactive motion
(a character being physically dragged by an animation) is destroyed before the base
teardown, because it holds both a shell and an animation and the base will invalidate the
animation. The shell leaves the dynamics world before it is freed.

## `activate_physic_shell`

**Contract** — the "this object just became a loose physical thing" path: a weapon
dropped, a corpse's gear thrown clear, an item knocked off a shelf. Requires that no shell
exists. Builds one, activates it between the object's current transform and a transform
displaced forward along the object's own forward axis — the dynamics seam uses the pair as
a swept hint so the activation does not tunnel through nearby geometry — then forces a
bone evaluation on this object's model and on its parent's, resolves any spawn penetration
(`correct_spawn_pos`), gives the body a launch velocity, and adopts the resulting pose.

```text
FUNCTION activate_physic_shell()
  REQUIRE shell IS none
  create_shell()                       # subclass decides the body layout
  launch <- forward_axis * 4           # see note
  shell.activate(from: transform, to: transform translated by launch)
  recompute_bones(parent_visual); recompute_bones(own_visual)
  IF multiplayer AND self IS NOT rocket AND self IS NOT grenade THEN
    shell.ignore_dynamic_bodies()      # dropped junk must not shove players around
  correct_spawn_pos()
  IF subclass overrides the launch speed THEN shell.velocity <- override
  ELSE shell.velocity <- launch
  transform <- shell.global_transform()
  recompute_bones(parent_visual)
  IF parent IS a shell holder THEN parent.on_child_shell_activate(self)
```

**Notes** — the bone recomputation happens twice on the parent, before and after: once so
that the attachment point the shell is released from is current, once so that the parent's
model no longer shows the departed object at a stale bone. This is the kind of duplicated
work that looks removable and is not.

The multiplayer exception for rockets and grenades is the point of the rule: thrown
ordnance *must* interact with players, everything else dropped must not, because a client
and the server would disagree about the shove and the player would rubber-band.

The launch magnitude is four units per second along the object's forward axis, but it
reads as an accident: the vector is scaled by two, twice, with an "up" vector computed
alongside it and never used at all. A rebuild should pick a launch speed deliberately and
write it once. **Could not recover**: whether four was chosen or arrived at.

## `setup_physic_shell`

**Contract** — the other acquisition path: an object that *is* physics from the moment it
appears, rather than one that becomes physics. Requires no existing shell. Builds it,
activates it in place (no swept hint, no launch velocity), recomputes the bones, applies
the object's own spawn-record configuration overrides to the shell, resolves penetration,
and adopts the pose.

The difference from `activate_physic_shell` is exactly the two things a *thrown* object
needs and a *placed* one must not have: a motion hint and a velocity.

## `correct_spawn_pos`

**Contract** — pushes a freshly activated body out of any geometry it was created inside.
Returns immediately, doing nothing, when the object's parent already declares a collision
place for it — an item leaving a holster is *meant* to overlap its owner.

Otherwise: measure the shell's bounding box, disable the shell's own collision, grow a
temporary probe body of that box at that place and let the dynamics seam resolve it out of
the surrounding world, re-enable collision, and translate the shell by the displacement the
probe found.

```text
FUNCTION correct_spawn_pos()
  REQUIRE shell EXISTS
  IF parent EXISTS AND parent.has_shell_collision_place(self) THEN RETURN
  (size, centre) <- bounding_box_of(shell, transform)
  REQUIRE size, centre and transform are all finite     # see note
  shell.disable_collision()
  resolved_centre <- run_activation_probe(self, transform, size, centre)
  shell.enable_collision()
  shell.translate_by(resolved_centre - centre)
  transform <- shell.global_transform()
```

**Notes** — the probe is a separate body rather than the shell itself because the shell may
be a multi-body assembly whose parts would resolve against each other; a single box cannot.
Disabling the shell's collision during the probe keeps the probe from resolving against the
very object it is finding room for.

The finiteness checks are not defensive noise: a model with a degenerate bone or a
zero-scale node produces a non-finite box, the probe then diverges, and the object is
launched to infinity — a failure that is far cheaper to catch here, with the object's name
and model in the message, than in the solver.

## `PHSaveState` / `PHLoadState`

**Contract** — write and read the full pose of a multi-body assembly, quantized. Used by
save games and by the network path for assemblies whose exact configuration matters
(ragdolls, broken skeletons).

```text
FUNCTION PHSaveState(out)
  IF visual is animated THEN
    out.write bone_visibility_mask : int (64-bit)   # one bit per bone
    out.write root_bone_index      : int (16-bit)
  ELSE
    out.write all-ones, 0                           # "everything visible, root is 0"
  box <- axis-aligned bounds over every element's position
  box <- box expanded by a small epsilon on every side   # see note
  out.write box.min, box.max                        # full precision, 3 reals each
  out.write element_count : int (16-bit)
  FOR EACH element IN elements
    element.state.write_quantized(out, box.min, box.max)
```

Loading is the mirror: restore the bone visibility mask and root, read the box, read the
count, and set each element's state from its quantized form against the same box.

**Invariants**

- The box must be non-degenerate — a zero-extent axis makes the quantization divide by
  zero. The epsilon expansion is what guarantees it for a single-body assembly, whose
  bounds are otherwise one point. That is the entire reason the expansion exists.
- The element order on load must be the element order on save. Nothing in the stream
  identifies elements; they are positional. This ties the saved state to the model's bone
  ordering, and is why a save refuses to load against a changed model.
- The bone visibility mask is 64 bits wide, which caps a saveable skeleton at 64 bones for
  this purpose. Shipped models stay under it.

**Notes** — the positions are quantized against a per-record bounding box rather than
against world coordinates, so precision is proportional to the assembly's own size rather
than to the level's. A ragdoll two metres across keeps millimetre precision in a level
kilometres wide. This is the reason for the box, not compactness.

## `save` / `load`

**Contract** — `save` appends the base's state and then one byte: whether the shell is
awake, asleep, or has no opinion (no shell, or an inactive one). `load` reads the base's
state and the byte.

**Notes** — and here is the file's worst decision. The byte read by `load` is stored in a
**process-wide variable**, not on the object, because at load time the object has no shell
to apply it to — the shell is built later, during spawn. `net_Spawn` reads the variable
back and clears it.

The invariant this demands is severe and written down nowhere: between one object's load
and that same object's spawn, no other shell holder may load. Restoring a save is
therefore strictly load-then-spawn per object, never load-all-then-spawn-all. A rebuild
should simply keep the enable state as a field on the object and delete the hazard; it
costs one byte per object and removes an ordering constraint that cannot be checked.

## `OnChangeVisual`

**Contract** — reacts to the object's model being replaced or cleared. When the model is
cleared, any interactive motion is destroyed and the shell is deactivated and freed,
because a shell's bodies are bound to bones that no longer exist. When a model is merely
replaced, the base's handling suffices.

## `Hit` / `PHHit`

**Contract** — `Hit` is the game-layer damage entry point; the base's only job with it is
to convert the hit's physical impulse into a push on the body, at the hit's bone-space
position, along its direction, attributed to its bone and damage type. Zero impulse or no
shell means no push.

One case is excluded: a burn hit an object delivers to *itself* with no bone named. A
burning creature applies fire damage to its own body every tick; letting that push the
body would make anything on fire skitter across the level. Damage still applies — only the
impulse is suppressed.

## `physics_collision`

**Contract** — returns the collision interface for this object. For a character, this
first ensures the *animation collision* exists — the lightweight body set that follows an
animated character's bones so that things can collide with it while it is under animation
control rather than under the solver's. Creating it lazily here means an object that is
never collided against never pays for it.

## `physics_shell` (const) / `physics_shell` / `physics_character`

**Contract** — three views of "what body does this object present to the world".

The const form has a fallback that matters: if there is no real shell, but the object is a
character with an animation collision, the animation collision's shell is returned. From
the outside, an animated character and a ragdolled one both have a body; the difference
between the two is this class's business, not the caller's.

The mutable form has no such fallback — a caller intending to *modify* the body must be
talking about a real, solver-owned shell.

`physics_character` returns the single body a character's movement control walks on, or
nothing for a non-character.

## `PHSetMaterial`

**Contract** — sets the surface material of every body in the assembly, addressed either
by name or by the material table's index. No-op without a shell. The material decides
friction, bounce, and which impact sound and particle effect a collision produces.

## `PHGetLinearVell` / `PHSetLinearVell` / `GetMass`

**Contract** — read and write the assembly's linear velocity and read its total mass. With
no shell, velocity reads as zero, writes are dropped, and mass reads as zero. The
zero-instead-of-error choice is deliberate: callers ask these of arbitrary objects.

## `PHGetSyncItemsNumber` / `PHGetSyncItem`

**Contract** — the network and save layers' view of the assembly as an indexed list of
synchronizable elements. Count is zero and every item is absent when there is no shell.
Index order is the assembly's element order, which is the ordering contract `PHSaveState`
depends on.

## `PHFreeze` / `PHUnFreeze`

**Contract** — force the assembly asleep or awake regardless of the solver's own sleeping
heuristic. No-op without a shell. Used when the game knows something the solver cannot:
an object entering an anomaly's grip, a level about to unload.

## `EffectiveGravity`

**Contract** — the gravity this object falls under, defaulting to the dynamics world's.
Virtual so that objects inside gravity anomalies, or with buoyancy, can answer differently
without the solver knowing anything about anomalies.

## `create_physic_shell`

**Contract** — asks the object itself, through the shell-creator interface, to build its
body layout. Requires that no shell exists. An object that does not implement the creator
interface silently ends up with no shell, which is legal.

## `deactivate_physics_shell`

**Contract** — removes the assembly from the dynamics world and frees it.

## `SheduleRegister` / `SheduleUnregister` / `IsSheduled` / `register_schedule`

**Contract** — idempotent registration and removal of this object from the engine's update
scheduler, tracked by a flag so that a double register or a double unregister is a no-op
rather than a corruption of the scheduler's list. `register_schedule` is how the engine
asks, at spawn, whether this object wants scheduled updates at all — the answer is simply
the current flag, which lets a subclass decide during its own spawn (see
[`PhysicsSkeletonObject.cpp`](PhysicsSkeletonObject.cpp.md), which drops off the
scheduler when its assembly cannot break).

## `UpdateCL`

**Contract** — per frame: the base's client update, then re-anchor every attached particle
emitter to its bone. Particles are attached to bones and the pose changed this frame.

## `has_shell_collision_place` / `on_child_shell_activate`

**Contract** — two questions the parent of an activating object gets asked: "do you
already have a place where this object's body is allowed to overlap yours?" (answered by a
character's physics support; non-characters say no) and "one of your children just became
physical" (a character reacts; others ignore it).

## `on_physics_disable`

**Contract** — called when the solver puts this object's body to sleep. Single player does
nothing. The multiplayer path that would have told the server about it is disabled, so
this is currently a hook with no behaviour behind it. **Could not recover**: whether sleep
is meant to be replicated and was removed as a bug, or was never finished.

## The physics-module adapter surface

**Contract** — the questions the dynamics module is allowed to ask a game object, each a
direct forward:

| It asks | The game layer answers with |
|---|---|
| `ObjectXFORM`, `ObjectPosition` | the object's transform and position, by reference, so the solver can write the pose back |
| `ObjectName`, `ObjectNameVisual`, `ObjectNameSect` | the instance name, model name and configuration section — diagnostics only |
| `ObjectID` | the entity identifier |
| `ObjectGetDestroy` | whether the object is already marked for destruction, so the solver can drop it |
| `ObjectCollisionModel` | the static collision form |
| `ObjectKinematics` | the animated model, required to exist |
| `ObjectCastIDamageSource` | a damage attribution source, or nothing |
| `ObjectGetCollisionHitCallback` | the per-object collision hit callback, or nothing |
| `ObjectPhSoundPlayer` | the collision sound player |
| `ObjectPhCollisionDamageReceiver` | the collision damage receiver, or nothing |
| `ObjectProcessingActivate` / `Deactivate` | enter and leave the engine's per-frame processing set |
| `ObjectSpatialMove` | the object moved; re-index it in the spatial database |
| `ObjectPPhysicsShell` | the shell slot itself, by reference |
| `has_parent_object` | whether the object is attached to another |
| `PHCapture` | the grab controller, if this is a character holding something |
| `IsInventoryItem`, `IsActor`, `IsStalker` | type queries, answered by attempted casts |
| `IsCollideWithBullets`, `IsCollideWithActorCamera` | both true at the base; subclasses opt out |
| `HideAllWeapons` | nothing at the base; a creature hides its weapons when ragdolled |
| `MovementCollisionEnable` | forwarded to the character's movement control, which must exist |
| `BonceDamagerCallback` | a character that is a "specific damager" replaces the collision damage multiplier |

**Notes** — the type queries are the interesting entry. The dynamics module needs three
coarse type distinctions (is this the player, is this a person, is this a carryable item)
to apply different contact rules, and getting them through this interface rather than
through the game's class hierarchy is what keeps the physics module compilable without the
game. A rebuild should replace them with explicit capability flags set at spawn — three
booleans on the body, which is what they cost anyway.

`IsCollideWithActorCamera` is the rule that stops the player's third-person camera being
shoved by props; objects that should be transparent to it answer false.

## `dump`

**Contract** — debug-only. Renders the object as text at one of six levels of detail
(identity, poses, visual geometry, properties, everything, everything truncated) for the
object inspector. Omitted from release builds entirely.
