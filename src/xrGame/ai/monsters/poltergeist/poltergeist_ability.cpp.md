# src/xrGame/ai/monsters/poltergeist/poltergeist_ability.cpp

> The shared half of a poltergeist's ability — the particle effects and idle sound every poltergeist has whichever ability it carries — plus the two ambient scares the creature performs directly.

**Needs** — [`poltergeist.h`](poltergeist.h.md) · [Seam: Rigid-body dynamics](../../../../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics) · [Seam: Static collision database](../../../../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database) · [Seam: Audio device](../../../../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: owns particle emitters and sound handles that must be destroyed at defined moments, and applies impulses to the rigid-body layer

## Purpose

Two unrelated things share this file.

The first is `PolterAbility`, the base both concrete abilities extend. It carries what is
common: **two simultaneous particle effects while the creature is hidden** — one for the
creature's presence and one for its electrical idle — a looping vocalisation that follows the
creature, a damage effect placed at the exact point a bullet struck, and a death burst. A
concrete ability inherits all of that and adds only its attack.

The second is the pair of ambient scares the creature performs itself: shoving a random nearby
physics object, and making an unexplained noise off a nearby surface. Neither is wired into any
state in the shipped game — see the Notes — but both are complete and both are the source of
the creature's name.

## `PolterAbility` state

```text
RECORD PolterAbility
  creature            : Poltergeist
  presence_particles  : optional<emitter>   # alive exactly while the creature is hidden
  idle_particles      : optional<emitter>   # likewise
  idle_sound          : sound               # looping, follows the creature

  # authored in the creature's configuration section — all four required
  particles_hidden    : text   # "Particles_Hidden"
  particles_damage    : text   # "Particles_Damage"
  particles_death     : text   # "Particles_Death"
  particles_idle      : text   # "Particles_Idle"
  sound_idle          : text   # "Sound_Idle", played as a monster vocalisation

  last_hit_frame      : int    # frame number of the last damage effect
```

Invariant: both emitters are non-empty exactly while the creature is hidden. The hidden-mode
transition is the only thing that creates or destroys them, which is why entering hidden mode
asserts that none exists.

## `update_schedule`

**Contract** — keeps the idle vocalisation alive at the creature's position while it is alive:
starts it if not playing, otherwise just moves it. A dead poltergeist falls silent without
anything explicitly stopping the sound.

## `on_hide` / `on_show`

**Contract** — `on_hide` starts both emitters at the creature's position, each with a slight
upward bias, and does nothing at all if the creature is already dead. `on_show` destroys both.
Neither touches the sound.

## `update_frame`

**Contract** — every frame, copies the creature's transform onto both emitters, so the effects
follow the drifting, invisible creature. This is the whole reason the emitters are held rather
than fire-and-forget.

## `on_die`

**Contract** — plays the death burst at the creature's **graph position lifted by its current
drift height** — that is, where the player last saw the effect, not where the body is — then
destroys both emitters.

## `on_hit`

**Contract** — places one damage effect at the exact point on the creature a bullet struck, at
most once per frame, and only for gunfire.

```text
FUNCTION on_hit(hit)
  IF creature is alive AND hit.type = firearm AND this is a new frame
    IF the hit names a bone
      point = hit.position_in_bone_space
      transform point by that bone's current pose, then by the creature's transform
      play the damage effect at point, biased upward
  last_hit_frame = current frame
```

**Notes** — the once-per-frame guard is what stops a shotgun's pellets producing a dozen
overlapping effects. The bone-space to world transform is the ordinary two-step through the
skeleton; what matters for a rebuild is that the effect is placed at the *impact point on the
model*, not at the creature's origin, so a poltergeist shows where it was hit even though it is
invisible. That is a deliberate tell.

Only firearm damage produces an effect; fire, psi and anomaly damage do not.

## `physical_impulse`

**Contract** — picks one random physics object within a radius of a position and shoves one of
its parts away from that position. Part of the creature, not of the ability.

```text
FUNCTION physical_impulse(position)
  candidates = every object within IMPULSE_RADIUS of position
  IF none THEN RETURN
  pick one at random
  IF it has no physics body THEN RETURN          # a single attempt; no retry

  direction = away from position, normalised
  part      = one of its physics parts, at random
  apply an impulse of IMPULSE times that PART's mass along direction
```

**Notes** — the candidate is chosen at random and then rejected if it has no physics body,
with no second attempt, so the effect fires on well under all of its invocations. That is
acceptable for an ambient scare and would not be for an attack.

The impulse scales with the *part's* mass rather than the whole object's — the source keeps the
whole-object version beside it, disabled. Scaling by part mass gives a constant velocity change
to that part regardless of how heavy the rest of the object is, so a heavy object is nudged and
a light one is flung. The two constants — an impulse factor of 10 and a radius of 5 world
units — are unexplained.

## `strange_sounds`

**Contract** — finds a nearby surface by casting rays in random directions, works out what
material it is, and plays one of that material's own collision sounds just short of the
surface. Does nothing if a strange sound is already playing.

```text
FUNCTION strange_sounds(position)
  IF a strange sound is still playing THEN RETURN

  REPEAT up to TRACE_ATTEMPTS times
    direction = uniformly random
    IF a ray from position along direction hits static geometry within TRACE_DISTANCE
      pair = the material pair of (this creature's material, the surface's material)
      IF no such pair THEN try the next direction
      IF the pair has collision sounds
        clone one of them at random
        play it just short of the hit point, pulled back a tenth of a unit
        RETURN
```

**Notes** — reusing the *material collision* sound bank is the trick that makes this effect
work without any assets of its own: the noise a poltergeist makes off a metal door is the noise
that door makes when something hits it. A rebuild that gives the creature its own sound bank
loses the property that the scare is always appropriate to the surroundings.

Three attempts, ten units of range, and the tenth-of-a-unit pullback are unexplained; the
pullback is plainly to keep the emitter out of the geometry.

## What is not wired up

Neither `physical_impulse` nor `strange_sounds` is called from any live path in the shipped
game. The only caller is the commented-out scare branch in
[`poltergeist_state_manager.cpp`](poltergeist_state_manager.cpp.md). A rebuild aiming at
fidelity must leave them unreachable; a rebuild aiming at the *intended* creature has both
implementations and a note saying where they were meant to be driven from.
