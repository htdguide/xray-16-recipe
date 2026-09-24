# src/xrGame/ai/monsters/scanning_ability_inline.h

> The movement-detection accumulator: how the score rises, how it decays, and the process-wide token that stops a roomful of scanners from stacking the same screen effect.

**Needs** — [`scanning_ability.h`](scanning_ability.h.md) · [`ai_monster_effector.h`](ai_monster_effector.h.md) · [Seam: Audio device](../../../../SYSTEM-REQUIREMENTS.md#seam-audio-device) · [Seam: Graphics device](../../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`scanning_ability.h`](scanning_ability.h.md)
**Tier floor** — T2: a scalar integrator over the player's measured speed

## Purpose

Three ideas, none of which the shipped engine ever runs (see
[`scanning_ability.h`](scanning_ability.h.md) for why): the detection integral, its two-rate
structure, and the shared-effect token.

## `load`

**Contract** — reads the whole ability from a configuration section: the five tuning numbers,
the sound, and a *named second section* holding the screen-effect description. Fails hard on a
missing key — every one is mandatory. Allocates the sound handle.

**Notes** — the screen effect lives in its own section named by the creature's section, which
is the pattern the whole chapter uses for post-process descriptions: the creature's data says
*which* effect, and a separate authored block says what the effect looks like. Three of the
effect's fields are colour triples stored as comma-separated text and parsed at load, which is
an artefact of the configuration format having no vector type, not a decision.

## `schedule_update` — the detection step

**Contract** — runs on the creature's rate-degraded update. Reads the player's position and
measured speed, advances the score, may start a sound and a screen effect, and may fire the
success hook. Does nothing when the ability is disabled, when the creature is dead, or when
the current camera entity is not the player.

```text
FUNCTION schedule_update()
  IF holds_the_token AND scan_sound HAS FINISHED
    release_token()                   # the effect is over; another scanner may claim it

  IF phase == disabled OR owner IS dead
    RETURN
  player = the entity the camera is attached to
  IF player IS none
    RETURN                            # only ever scans the player, never another creature

  IF phase == armed AND distance(player, owner) < scan_radius
    phase = scanning                  # crossing in arms it; crossing out again does NOT disarm

  IF phase == scanning
    speed = player.measured_speed
    IF speed > velocity_threshold
      IF now > last_add_at + (1000 / scan_trace_time_freq)
        last_add_at = now
        score = score + speed         # the faster the player moves, the more one sample is worth

      IF scan_sound IS PLAYING
        scan_sound.follow(player.position)
      ELSE IF token_is_free
        scan_sound.play_at(player.position)
        push_screen_effect(effector)
        claim_token()
      on_scanning()

  IF score > critical_value
    on_scan_success()
    phase = disabled                  # one-shot: firing ends the ability until re-enabled
```

**Invariants**

- **Arming is one-way.** Once the player has come within the radius, leaving it does not put
  the ability back to armed — the score keeps accumulating whenever the player moves, at any
  distance. The radius is a trigger, not a gate. Whether that was intended is not recoverable,
  but it is what the code does, and a rebuild reproducing "you can walk out of range to be
  safe" would be reproducing something else.
- **Score additions are rate-limited, not per-tick.** The interval is the reciprocal of the
  authored frequency, so the score's growth rate is independent of how often the scheduler
  happens to run this creature. That is the reason the frequency is authored rather than
  derived: without it a nearby creature would detect faster than a distant one purely from
  scheduling.
- **One sample adds the player's whole speed**, not a fixed quantum, so sprinting past is much
  more revealing than walking past — the score is an integral of speed, not of time.
- **The sound's lifetime is the effect's lifetime.** The token is released when the sound
  finishes, not on a timer, which couples the visual duration to the audio asset's length.

**Notes** — the token is a single flag shared by every instance of the owning creature type,
which is how a pack of scanners produces one screen effect rather than one per creature. The
source marks it as provisional ("make this postprocess with a static check, only one for all
scanners"), and it is: the flag lives on the creature type rather than on the effect system, so
two *different* scanning creature types would each get their own token and would stack. A
rebuild should own the exclusion in whatever applies the screen effect.

## `frame_update` — the decay step

**Contract** — runs every frame, takes the frame's elapsed milliseconds. Subtracts the authored
decay rate scaled by elapsed time, clamped at zero. Only runs while scanning.

**Notes** — decay is per frame and accumulation is per scheduled update, and that asymmetry is
deliberate: decay must be smooth and frame-rate independent for the mechanic to feel fair,
while accumulation must not depend on how often the creature is scheduled. A rebuild that puts
both on the same clock will get a detection rate that varies with distance.

## `get_velocity`

**Contract** — the player's actual measured movement speed, taken from the physics movement
state rather than from a per-frame position difference, so that being pushed, falling or
sliding all count as movement.

## `enable`, `disable`, `on_destroy`

**Contract** — `enable` moves a *disabled* ability to armed and clears the score; on an already
active ability it does nothing, so re-enabling cannot be used to reset the score mid-scan.
`disable` returns to disabled and clears the score unconditionally. `on_destroy` releases the
shared effect token if this instance holds it — without it, a creature killed mid-scan would
leave the token claimed forever and no scanner would ever show the effect again.
