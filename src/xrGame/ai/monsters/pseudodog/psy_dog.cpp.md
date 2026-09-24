# src/xrGame/ai/monsters/pseudodog/psy_dog.cpp

> A dog that fights by proxy: it keeps a population of illusory copies of itself alive, hides while too few are up, and each copy spawns unseen, materialises in the player's face with a leap and a jolt, and dissolves at the first hit from its target.

**Needs** — [`psy_dog.h`](psy_dog.h.md) · [`pseudodog.h`](pseudodog.h.md) · [`psy_dog_aura.h`](psy_dog_aura.h.md) · [`psy_dog_state_manager.h`](psy_dog_state_manager.h.md) · [`../ai_monster_effector.h`](../ai_monster_effector.h.md) · [`../../../xrAICore/Navigation/level_graph.h`](../../../../xrAICore/Navigation/level_graph.h.md) · [Seam: Networking transport](../../../../../SYSTEM-REQUIREMENTS.md#seam-networking-transport) · [Seam: Graphics device](../../../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`psy_dog.h`](psy_dog.h.md)
**Tier floor** — T2: spawns entities into the world at runtime through the authoritative record path and destroys them through the same

## Purpose

The psi dog is the chapter's most structurally interesting creature because it is the only one
whose behaviour is *another set of creatures*. Four decisions make it.

**The phantoms are real entities.** They are not effects or projections: each is spawned from a
configuration section as a full creature with its own body, navigation, animation and
perception, into the authoritative record stream, exactly as an authored spawn would be. Which
means each can path, leap, and be hit — and also that the mechanic costs what four creatures
cost.

**The parent hides behind the population.** The psi dog's brain asks one question — are there
fewer phantoms up than the authored minimum — and if so, drops everything and hides. So killing
the phantoms is what forces the real dog into the open, and that is the encounter.

**A phantom is one hit.** Any damage from the entity it is hunting destroys it instantly,
ignoring health entirely. The player cannot tell a phantom from the real dog by looking, only
by shooting it.

**The respawn schedule is per slot, not per population.** The parent keeps an array with one
timestamp per possible phantom, and each slot independently becomes eligible again a fixed
delay after the phantom that occupied it died. Killing all four at once therefore brings all
four back together; killing them one at a time brings them back one at a time.

## State

```text
RECORD PsyDog                      # on top of PseudoDog
  aura            : PsyDogAura
  phantoms        : list<PsyDogPhantom>   # the live ones
  min_phantoms    : int      # authored "Min_Phantoms_Count", default 1 — the hiding threshold
  max_phantoms    : int      # authored "Phantoms_Count" or "Max_Phantoms_Count"
  respawn_delay   : int      # authored "Time_Phantom_Respawn" or "Time_Phantom_Appear"
  slot_died_at    : list<int>             # exactly max_phantoms entries; the respawn schedule

RECORD PsyDogPhantom               # also a PseudoDog
  parent          : optional<PsyDog>
  parent_id       : int          # the identity handed down in the spawn record; NONE = destroying
  phase           : enum { waiting_to_appear, attacking }
  spawned_at      : int
  appear_effect   : screen-and-camera effect parameters   # authored "appear_effector"
  particles_appear, particles_disappear : text            # authored
```

Two sentinel values sit in the slot array and are not times: one means *eligible immediately*
and one means *this slot is occupied*. Both are tiny integers, so a real timestamp can never
collide with them.

## `Load`

**Contract** — reads the aura's screen-effect section by name, the population bounds and the
respawn delay, and sizes the slot array to the maximum with every slot marked immediately
eligible. The maximum and the delay are each read under **two alternative key names**, the
second being the older spelling, because the shipped data files disagree between games.

**Notes** — starting every slot eligible means a psi dog that acquires an enemy fills its whole
population on the first tick, rather than staggering the first wave. Only replacements are
staggered.

## `think` — the population loop

**Contract** — the parent's per-tick step. Drives the aura, then either tops the population up
or wipes it.

```text
FUNCTION think()
  base creature's thinking
  IF dead THEN RETURN

  aura.update_schedule()

  IF there is an enemy AND phantom_count < max_phantoms
    FOR EACH slot IN slot_died_at
      IF the slot is not marked occupied
         AND now > slot_died_at[slot] + respawn_delay
        IF spawn_phantom() succeeded
          mark the slot occupied
  ELSE IF there is no enemy AND any phantoms are up
    destroy_all_phantoms()
```

**Invariants** — phantoms exist only while the parent has an enemy. Losing the enemy wipes the
whole population at once, which is what makes a psi dog encounter end cleanly.

**Notes** — the loop can fill several slots in one tick, so the delay staggers *replacement of
a given slot*, not the rate of spawning overall.

A slot is marked occupied on a successful spawn but is only re-stamped with a death time when a
phantom unregisters. The association between a slot and the phantom occupying it is by *index
position in the live list at the moment of unregistration*, which is not the same index as the
slot that spawned it. The mapping is therefore loose: with more than one phantom up, a death
can re-arm a different slot than the one the dead phantom came from. Since all slots carry the
same delay, the visible behaviour is unaffected; a rebuild should nevertheless give each
phantom its slot index and stop guessing.

## `unregister_phantom`

**Contract** — removes a phantom from the live list and stamps the corresponding slot with the
current time, arming its respawn.

**Notes** — the original computes the slot index from an iterator *after* erasing through it,
which reads the removed position. The index it obtains is the position the phantom occupied,
which is what is wanted, but it is obtained from an invalidated handle. A rebuild should
capture the index before removing. This is the one place in the file where the original is
relying on unspecified behaviour rather than merely being loose.

## `spawn_phantom`

**Contract** — finds a navigable vertex four to eight units from the parent, creates a spawn
record for the phantom section at that vertex, stamps the parent's identity into the record's
special-object field, and publishes it into the authoritative record stream. Answers whether a
vertex was found.

```text
FUNCTION spawn_phantom() -> bool
  IF no reachable vertex exists in the band [4, 8] around the parent THEN RETURN false

  section = the parent's own "phantom_section", defaulting to the standard phantom
  record  = create a spawn record for that section at the vertex
  record.special_object = the parent's identity          # this is the whole handshake
  publish the record
  RETURN true
```

**Invariants** — the parent's identity travels to the child through the **spawn record's
special-object field**, which is the same field authored spawns use to bind a creature to a
thing it belongs to. That is the only channel: the parent does not hold the child until the
child finds it.

**Notes** — the band of four to eight units is close enough that the phantoms appear as a pack
around the real dog and far enough that they do not spawn inside it. Five samples. Neither
number is derived.

The phantom's section is read from the parent's own configuration, defaulting to a standard
name, so a modded psi dog can conjure something other than a copy of itself.

## The phantom's spawn

**Contract** — on spawning, a phantom reads its parent's identity out of its own spawn record,
tries immediately to find that parent, and makes itself **invisible and inert** — not rendered,
not updated by the engine's usual path. It loads its appear effect and its two particle names,
and stamps its spawn time.

**Notes** — spawning invisible is what makes the materialisation a moment rather than a pop.
An identity of all-ones in the parent field is the sentinel for "this phantom is in the process
of destroying itself" and is checked before every other path.

## The phantom's `think`

**Contract** — the phantom's per-tick step: find the parent if not yet found, check the two
self-destruct conditions, inherit the target, and — once facing it — materialise.

```text
FUNCTION think()
  IF already destroying THEN RETURN
  base creature's thinking
  try_to_find_parent()

  IF parent found AND distance to parent > 30        THEN destroy_me() ; RETURN
  IF parent not found
    IF spawned_at + PARENT_WAIT > now THEN destroy_me()      # see Notes
    RETURN

  IF phase is not waiting_to_appear THEN RETURN

  inherit the parent's current enemy into my own memory

  IF I have an enemy AND I am not facing it within a thirtieth of a turn THEN RETURN

  leap_target = 10 units straight ahead
  temporarily add a movement border between here and there
  vertex = the last navigable vertex walking towards leap_target
  remove the border
  IF that vertex is valid and reachable
    leap at the target, lifted by one unit

  phase = attacking
  become visible and active
  start the appear particles

  IF my enemy is the player
    jolt the player's camera and screen with the authored appear effect
```

**Invariants** — a phantom never materialises facing away from its target. The facing tolerance
is a thirtieth of a turn, which is tight: the phantom waits, turning, until it is essentially
aimed at the player, and then appears already lunging.

**Notes** — the parent-wait condition is inverted. Its comment says *destroy me if there has
been no parent for a long time*, and the test destroys the phantom while it is still **inside**
the waiting window, so a phantom that does not find its parent on the very first tick kills
itself immediately rather than waiting. In practice the parent is always found at once, so the
path is unreachable; reproduce it as written and flag it.

The leap is attempted but not required — if the ground ahead is not navigable the phantom
materialises without leaping. That is why phantoms appear both lunging and standing.

The movement border added around the leap line, and removed straight after, constrains the
navigation query to a corridor so the walk cannot wander around an obstacle. The
add-query-remove shape is the idiom; a rebuild should pass the constraint as a query parameter.

The camera and screen jolt fires only when the target is the player, because it is a player-
facing effect and a phantom hunting another creature has nobody to jolt.

## The phantom's destruction paths

Three of them, and the distinction matters.

**`on_hit`** — any hit whose source is this phantom's own target destroys it. Not damage, not
health: the identity of the attacker. Hits from anything else are ignored entirely, so
phantoms are immune to crossfire, explosions and each other. This is the mechanic.

**`destroy_me`** — the phantom's own decision: unregister from the parent, mark itself as
destroying, and publish a destroy event for itself.

**`destroy_from_parent`** — the parent's decision: mark as destroying *without* unregistering,
then publish the same destroy event. The parent is already clearing its own list, so
unregistering would be a double removal.

All three converge on a destroy event through the authoritative record stream rather than an
immediate deletion, so the removal is ordered with everything else in the world.

**`on_despawn`** — plays the disappear particles at the phantom's centre and unregisters if it
had not already. This runs for every destruction path, which is why the visual is here and not
in any of the three.

## `create_state_manager`

**Contract** — substitutes the psi dog's brain, described in
[`psy_dog_state_manager.cpp`](psy_dog_state_manager.cpp.md), over the pseudodog's body. This
one line is the whole of the "psi dog is a pseudodog plus a state" claim.

## `on_death` / `on_despawn`

**Contract** — both destroy every phantom; death additionally shuts down the aura. A psi dog's
phantoms never outlive it.
