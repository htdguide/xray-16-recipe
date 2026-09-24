# src/xrGame/DestroyablePhysicsObject.cpp

> A physics prop that breaks: it accumulates damage through the armour and per-bone scaling tables, and on reaching zero replaces itself with a pre-authored broken version, with a sound and a burst of particles oriented to the blow.

**Needs** — [`DestroyablePhysicsObject.h`](DestroyablePhysicsObject.h.md) · [`PhysicObject.h`](PhysicObject.cpp.md) · [`PHDestroyable.h`](PHDestroyable.h.md) · [`PHCollisionDamageReceiver.h`](PHCollisionDamageReceiver.h.md) · [`hit_immunity.h`](hit_immunity.h.md) · [`damage_manager.h`](damage_manager.h.md) · [`Hit.h`](Hit.h.md) · [`script_game_object.h`](script_game_object.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — reached through its declarations in [`DestroyablePhysicsObject.h`](DestroyablePhysicsObject.h.md); callers name that, not this file.
**Tier floor** — T2: damage arithmetic and a model swap; the physics is behind the seam

## Purpose

The breakable prop. Its notable property is that *everything about how it breaks is
authored into the model file*, not into the configuration ltx: the model carries a block
of user data naming the broken version, the damage scaling per bone, the immunity table,
the break sound, the particle effect, and even which objects are allowed to damage it at
all. So a level artist makes a crate breakable without touching code or configuration —
which is why this class is generic and has no per-prop subclasses.

Destruction is not deletion. The object swaps to a pre-authored broken model whose pieces
are separate physics bodies, and keeps living until the sound and the particles have
finished. That deferral is the second load-bearing decision: an object that vanished the
instant it broke would cut its own effects off.

## State

```text
RECORD DestroyablePhysicsObject
  health            : real     # starts at 1; damage is subtracted from it directly
  destroy_sound     : sound
  destroy_particles : text
  allowed_attackers : list<text>   # object names; empty means "anyone"
```

Invariants: health starts at one and is in the same units as a post-scaling hit, so all
the tuning lives in the immunity and bone-scaling tables rather than in a health number.
The object may not be removed while its sound or particles are still running.

## `net_Spawn`

**Contract** — spawns the prop, then configures every damage-related subsystem from the
*model's* embedded user data. The order is load-bearing and each step is conditional on
its block existing, so a model may opt into any subset.

```text
FUNCTION spawn(record)
  base.spawn(record)
  data = the visual model's embedded configuration

  initialize the destructible substrate
  IF data has a "destroyed" block THEN
    load the broken-model description from it
  load the per-bone damage scaling from data's damage block
  IF data has an "immunities" block   THEN load the per-damage-type immunity table
  initialize collision damage reception          # being hit by other physics bodies
  IF data has a "sound" block         THEN create the break sound
  IF data has a "particles" block     THEN remember the break effect's name
  IF data has a "hit_from" block      THEN
    allowed_attackers = the keys of that block   # keys, not values
  load the model's particle attachment points
  play the model's startup animation, if the spawn record asks for one
```

**Notes**

- A model with no "destroyed" block still initializes the destructible substrate and is
  therefore breakable — it simply has nothing to become. The original's assertion
  demanding the block is commented out, so this is tolerated rather than intended.
- The attacker whitelist is read from the *keys* of its block, so the authored lines are
  `<object name> = <anything>`. The values are ignored.

## `Hit`

**Contract** — the damage path, and the order here is the whole contract. The script
callback fires *first and unconditionally*, before any filtering, so a script sees every
blow including ones the object will ignore. Then the attacker whitelist, then the immunity
table, then the per-bone scale, then the base's own handling (which applies the impulse to
the body), and only then the health subtraction.

```text
FUNCTION hit(blow)
  fire the script hit callback(self, power, direction, attacker, bone)

  IF allowed_attackers is non-empty
     AND attacker's name NOT IN allowed_attackers THEN RETURN    # immune to this attacker

  blow.power = blow.power * immunity_factor(blow.damage_type)
  blow.power = blow.power * bone_hit_scale(blow.bone)
  base.hit(blow)                       # impulse, wound effects, propagation

  health = health - blow.power
  IF health <= 0 THEN
    record this blow as the fatal one
    IF the destructible substrate is ready THEN destroy()
```

**Invariants** — the fatal blow is recorded before destruction runs, because the break
effect's orientation and the death callback's attacker both come from it.

**Notes** — the whitelist returns *after* the script callback and *before* the base hit, so
a whitelisted-out blow still reaches scripts but applies no impulse. A crate that only a
specific attacker can break is unmoved by every other blow, which reads oddly but is what
the shipped data expects.

## `Destroy`

**Contract** — performs the break. Fires the script death callback naming the attacker,
swaps the object to its broken form under a fixed class name, plays the break sound at the
object's position, and starts the break particles in a frame derived from the fatal blow's
direction. Finally re-registers with the update scheduler, because a broken object's
pieces need updating even though the intact object did not.

The particle frame construction is the interesting part:

```text
FUNCTION break_effect_frame(hit_direction)
  up = world up
  IF hit_direction is (anti)parallel to up THEN
    replace hit_direction with a random direction that is not
  # now build an orthonormal frame with up as its second axis and the blow in its plane
  i = up cross hit_direction, normalized
  k = i cross up
  RETURN frame (i, up, k)
```

**Invariants** — the effect is always upright (its second axis is world up) but yawed to
face the blow. A blow straight down or straight up leaves the yaw undefined, hence the
random substitution; without it the cross product degenerates and the effect is emitted
with a zero-length axis.

**Notes** — breaking must not happen while the physics world is mid-step, because it
deletes and creates bodies. The original asserts this rather than deferring, so the damage
path is required to run outside the solve.

## `InitServerObject`

**Contract** — writes the object's state back into its authoritative record, choosing
between two paths: a *copy* spawned as debris from an already-broken parent serializes as
an ordinary physics object, while an original serializes through the destructible
substrate, which records whether it has broken and what state its pieces are in. Either
way the record is tagged as a skeleton-type physics object.

**Invariants** — this is what makes destruction survive a save: the broken state is part of
the server record, so a reloaded level has the crate still broken and its pieces where
they fell.

## `CanRemoveObject`

**Contract** — refuses removal while the break particles are still playing or the break
sound still has a live voice. This is the deferral described in the Purpose, and it is the
reason the object re-registers with the scheduler when it breaks: something has to keep
asking.

## `OnChangeVisual`

**Contract** — tears down the physics shell before the visual changes, because the shell's
bodies are built from the old skeleton and would otherwise outlive it. The base then
builds a shell for the new visual.

**Invariants** — after the teardown the object must genuinely have no visual, which the
original asserts. A rebuild must order these the same way: shell first, then model.

## `net_Destroy` / `shedule_Update` / `_construct`

**Contract** — teardown resets the destructible substrate's respawn bookkeeping and clears
the collision-damage accumulator, so an object reused from a pool starts intact. The
scheduled update ticks the destructible substrate alongside the base. Construction
initializes the damage manager before the base, because the base's construction may
already consult it.

## Could not recover

- The starting health is a compiled-in 1.0 and is not read from anywhere, so a prop's
  toughness is expressible only through its immunity and bone-scale tables. Whether that
  was intended or simply never got a configuration key is not recorded.
- The fixed class name the broken form is spawned under is a magic string shared with the
  spawn factory; nothing in this file explains why it is not derived from the original
  object's own class.
