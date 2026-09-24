# src/xrGame/ai/monsters/monster_aura.cpp

> A proximity field around a creature that reaches the player directly: a strength falling off with distance, driving a screen effect, a looping sound whose volume tracks it, and a detector tick whose period shortens as the player closes.

**Needs** — [`monster_aura.h`](monster_aura.h.md) · [`basemonster/base_monster.h`](basemonster/base_monster.h.md) · [Seam: Audio device](../../../../SYSTEM-REQUIREMENTS.md#seam-audio-device) · [Seam: Graphics device](../../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`monster_aura.h`](monster_aura.h.md)
**Tier floor** — T2: owns a screen-effect registration on the player that must be released at a defined moment, and two sound handles

## Purpose

Some creatures affect the player without touching him. This is the mechanism: a scalar
*strength* computed from one distance, and three consumers of it.

The aura is aimed at exactly one target — the player — not at entities in general. That is the
single most important thing to know about it: the distance in every formula is the distance to
the player, the sounds are played at the player's head in two dimensions so they do not pan,
and the screen effect is attached to the player's camera. A creature's aura does nothing to
another creature.

Several auras can live on one creature, distinguished by a name that prefixes every
configuration key. A creature with a psi aura and a radiation aura reads
`psi_max_power`, `radiation_max_power` and so on out of one section.

## State

```text
RECORD MonsterAura
  creature            : BaseMonster
  name                : text                 # key prefix; fixed 64-byte buffer

  # authored, all optional — every one defaults to zero or absent
  linear_factor       : real                 # "<name>_linear_factor"
  quadratic_factor    : real                 # "<name>_quadratic_factor"
  max_power           : real                 # "<name>_max_power", the ceiling on strength
  max_distance        : real                 # "<name>_max_distance", beyond which strength is zero
  enable_for_dead     : bool                 # "<name>_enable_for_dead", whether a corpse still projects
  effect_name         : optional<text>       # "<name>_pp_effector_name", the screen effect to attach
  effect_full_at      : real                 # "<name>_pp_highest_at", the strength at which the effect is full
  loop_sound          : optional<sound>      # "<name>_sound"
  detect_sound        : optional<sound>      # "<name>_detect_sound"

  enabled             : bool                 # any of the above was present
  effect_slot         : int                  # 0 means not attached; otherwise the player's effect slot
  detect_accumulator  : real                 # seconds since the last detector tick
```

Invariants: `effect_slot` is non-zero exactly while a screen effect is attached to the player,
and it must be released before the aura is destroyed or the player keeps the effect forever —
teardown exists for that reason alone. `enabled` is set if *any* of effect name, max power,
max distance, or either sound was authored; an aura whose section mentions none of them is
inert and every per-tick path returns immediately.

## `strength`

**Contract** — the field's power at the player's current position, clamped to `max_power`.
Zero beyond `max_distance`. Pure; reads the player's position.

```text
FUNCTION strength() -> real
  distance = straight_line(creature.position, player.position)

  IF distance > max_distance THEN RETURN 0

  IF distance < EPSILON                      # standing inside the creature
    RETURN max_power IF either factor is non-zero ELSE 0

  power = linear_factor / distance + quadratic_factor / distance * distance
  RETURN min(power, max_power)
```

**Notes** — the second term reads as an inverse-square contribution and is named as one, but
as written the division and the multiplication cancel, so it contributes a flat
`quadratic_factor` at every distance inside the range. This is almost certainly not what was
meant; it is also what the shipped creatures were tuned against, so a rebuild that "fixes" it
changes the balance of every aura in the game. Reproduce the behaviour, and flag it.

The near-field special case exists because both terms diverge at zero distance. Returning the
ceiling there is the only sensible answer; returning zero when neither factor was authored
keeps an aura that only plays a sound from reporting full power at contact.

Every authored number passes through a debug override hook, which in a release build is the
identity. That hook is incidental — a rebuild needs it only if it wants live tuning.

## `load_from_ini`

**Contract** — reads the whole parameter set from a section, each key formed by concatenating
this aura's name with a fixed suffix. Every key is optional; absent keys take the defaults in
the record above. Creates the two sound resources if named. Sets `enabled` if anything at all
was found. `enable_for_dead` takes a caller-supplied default so a creature can decide whether
its kind of aura normally survives death.

## `update_schedule`

**Contract** — the per-tick step. Detaches the screen effect and returns when the aura should
not be working. Otherwise keeps the looping sound alive at the player's head with its volume
tracking the field, and attaches or detaches the screen effect around a threshold.

```text
FUNCTION update_schedule()
  IF NOT working()                     # dead creature without enable_for_dead, inert aura,
    detach_effect()                    # no player, or a dead player
    RETURN

  factor = effect_factor()

  IF the loop sound is not playing
    start it, looping, at the player's head, two-dimensional
  set its volume to factor

  IF no effect_name THEN RETURN

  IF factor > 0.01
    IF not attached
      effect_slot = player.reserve_effect_slot()
      attach effect_name at that slot, reading its intensity from effect_factor each frame
  ELSE IF attached
    detach_effect()
```

**Notes** — the attach threshold is a small epsilon rather than zero, so an aura the player is
barely inside does not thrash between attached and detached at the outer edge of its range.
The effect is attached with a *callback*, not a value: the renderer asks the aura for its
intensity every frame, which is why the per-tick rate of this routine does not make the effect
step visibly.

The loop sound is started and volume-tracked whether or not there is a screen effect, so an
aura may be sound-only.

## `effect_factor`

**Contract** — the field strength normalised against `effect_full_at` and clamped to the unit
interval. This is what the screen effect and the sound volume both consume, so that a field
can be tuned to saturate the screen well before it reaches its own power ceiling.

**Notes** — the debug override is applied to `effect_full_at` and then the *un-overridden*
field is used as the divisor, so live-tuning that number has no effect. Harmless in a release
build; a rebuild should divide by the same value it read.

## `play_detector_sound`

**Contract** — a discrete tick, like a Geiger counter, whose period shortens as the player
closes on the creature. Called by whatever device the player is carrying, not by the aura's own
update, so an aura ticks only while the player holds something that listens for it.

```text
FUNCTION play_detector_sound()
  IF NOT working() THEN RETURN

  distance = straight_line(creature.position, player.position)
  IF distance >= max_distance THEN RETURN

  power    = effect_factor()
  relative = distance / max_distance
  freq     = relative * (max_power / (power if non-zero else 1))
  IF relative > 0.65 THEN freq = freq / 2        # far out, slow the ticking further
  period   = 0.1 + 1.9 * freq                    # seconds, from a tenth up to two

  IF detect_accumulator > period
    play one detector tick at the player's head, two-dimensional
    detect_accumulator = 0
  ELSE
    detect_accumulator = detect_accumulator + frame_seconds
```

**Notes** — the period runs from a tenth of a second at the field's centre up to two seconds at
its edge; those two bounds are the whole character of the sound and are the constants to keep.
The extra halving past two-thirds of the range is a deliberate second gear so that the tick
does not creep up gradually from the very edge — it stays sparse and then quickens.

Dividing by the normalised strength means a field that saturates early ticks fast over most of
its range; the guard against a zero divisor makes an aura with no strength tick purely on
distance.

## `on_monster_death`

**Contract** — stops both sounds. Does *not* detach the screen effect, because an aura with
`enable_for_dead` set keeps working on a corpse and `update_schedule` remains the thing that
decides. The sounds are stopped unconditionally, so a corpse's aura goes silent while its
screen effect persists.
