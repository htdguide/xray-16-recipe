# src/xrGame/ai/monsters/scanning_ability.h

> A creature ability that detects the player by *movement* rather than by sight or sound — it accumulates a score while the player moves nearby, fires once when the score crosses a threshold, and then switches itself off. Declared here, implemented in the companion, and mixed into no creature.

**Needs** — [`scanning_ability_inline.h`](scanning_ability_inline.h.md)
**Used by** — [`burer.h`](burer/burer.h.md) · [`scanning_ability_inline.h`](scanning_ability_inline.h.md)
**Tier floor** — T2: a scalar accumulator with a decay rate and a distance gate

## Purpose

The interesting idea here is a sense that is neither vision nor sound: the creature has a
detection score that rises while the player is inside a radius and moving faster than a
threshold, and falls continuously otherwise. Crossing a critical value fires a one-shot hook
and disables the ability. Because the score both rises and decays, the player can defeat it by
standing still or by moving slowly — a stealth mechanic expressed entirely as one accumulator,
with no perception cone and no raycast.

**It is dead.** The burer is the only creature whose header includes this file, and the burer
does not inherit from it; nothing else mentions the type. The one trace it left behind is a
process-wide flag on the burer (see below) that is now written by nobody. A state identifier
for the corresponding behaviour exists and is even registered by the burer's brain, but the
ability that would select it does not run. Read this as a design, not as behaviour.

## State

```text
RECORD ScanningAbility
  owner                : reference to the creature
  # authored, all from the creature's configuration section
  critical_value       : real     # score at which the ability fires
  scan_radius          : real     # how close the player must come for scanning to begin
  velocity_threshold   : real     # how fast the player must move to add to the score
  decrease_value       : real     # score lost per second, always
  scan_trace_time_freq : real     # score additions per second; must not be zero
  scan_sound           : sound handle
  effector             : screen-effect parameters and its three durations
  # runtime
  phase                : ENUM { disabled, armed, scanning }
  score                : real
  last_add_at          : int      # clock reading of the last score addition
  holds_the_token      : bool     # this instance is the one currently playing the effect
```

**Invariants** — `scan_trace_time_freq` is asserted non-zero at load, because the additions
interval is its reciprocal. `score` never goes below zero. `phase` moves
disabled → armed → scanning in one direction only; firing returns it to disabled, and only an
explicit enable re-arms it.

## The demands on a filling

**`on_scan_success`** — fires once, when the score crosses the critical value. The ability
disables itself immediately after, so a filling that wants repeated detection must re-enable.

**`on_scanning`** — fires on every tick where the player is contributing to the score. Meant
for a per-tick tell (a sound, a posture) distinct from the detection itself.

The creature is also expected to carry a process-wide "the screen effect is free" flag, which
is how the ability avoids stacking one effect per creature; see
[`scanning_ability_inline.h`](scanning_ability_inline.h.md).

## Exported units

- `init_external` — bind to the owning creature.
- `load` — read every tuned number from a configuration section.
- `reinit` — clear the runtime fields; the ability starts disabled.
- `enable` / `disable` — arm and shut down.
- `schedule_update` — the detection step, on the creature's rate-degraded update.
- `frame_update` — the decay step, per frame.
- `on_destroy` — release the shared effect token.
