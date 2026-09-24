# src/xrGame/first_bullet_controller.cpp

> The multiplayer "first shot is accurate" rule: a player who has held fire for long enough and is moving slowly enough gets one shot at a reduced dispersion.

**Needs** — [`first_bullet_controller.h`](first_bullet_controller.h.md) · [`Level.h`](Level.h.md)
**Used by** — [`first_bullet_controller.h`](first_bullet_controller.h.md)
**Tier floor** — T3: two comparisons against a timestamp and a speed

## Purpose

Automatic weapons in a deathmatch reward holding the trigger; this rule rewards the opposite.
Wait, stop moving, and the first round goes where you aimed it. It exists only in
multiplayer — the single-player game has no equivalent and the predicate refuses to be asked
there at all.

The rule is small but it is the *whole* mechanism: nothing else in the engine decides that a
shot is a first bullet, and two separate systems consume the answer — the weapon substitutes
this dispersion for its own, and the damage pipeline looks up a per-bone first-bullet
multiplier (see [`damage_manager.cpp`](damage_manager.cpp.md)).

## State

```text
RECORD FirstBulletController
  use_first_bullet    : bool      # the whole feature, per weapon section
  fire_dispertion     : real      # the dispersion substituted for a qualifying shot
  shot_timeout        : int (ms)  # idle time the privilege requires
  actor_velocity_limit: real      # speed at or below which the privilege applies
  last_shot_time      : int (ms)  # global clock at the last shot, 0 meaning "never fired"
```

Invariant: when the feature is disabled, the other four values are whatever they were
initialised to and are never read. The load path leaves them untouched rather than assigning
defaults, so a rebuild must keep the feature flag as the gate on every use, not merely on
loading.

## `load`

**Contract** — reads the rule from a weapon's configuration section. The enable flag is
optional and defaults to off; if it is off, nothing else is read. If it is on, all three
tunables are **required** — a section that enables the feature without them is a data error
and fails loudly.

```text
FUNCTION load(section)
  use_first_bullet = config.optional_bool(section, "use_first_bullet", false)
  IF NOT use_first_bullet: RETURN
  fire_dispertion      = config.required_real(section, "first_bullet_dispertion")
  shot_timeout         = config.required_int (section, "first_bullet_timeout")
  actor_velocity_limit = config.required_real(section, "first_bullet_velocity_limit")
```

**Notes** — optional flag, required parameters. That asymmetry is the right shape for a
feature most weapons do not have: silence means off, and a half-configured weapon is caught
at load rather than behaving strangely in a match.

## `is_bullet_first`

**Contract** — the predicate, asked by the weapon at the moment of firing. Takes the
shooter's current linear speed. Yields false when the feature is off, when the shooter is
moving faster than the limit, or when the timeout has not elapsed since the last shot.
Refuses outright — a hard failure, not a false — when asked in single player.

```text
FUNCTION is_bullet_first(shooter_speed) -> bool
  IF single-player: FAIL WITH "first bullet shot can't be in single game mode"
  IF NOT use_first_bullet:                 RETURN false
  IF shooter_speed > actor_velocity_limit: RETURN false
  RETURN last_shot_time + shot_timeout <= global_clock
```

**Notes** — the single-player refusal is an assertion about *where this feature may be
reached from*, not a runtime branch. The rule changes weapon accuracy, and allowing it to
leak into the single-player game would silently alter the balance of the shipped campaign. A
rebuild should keep it as a hard boundary.

A never-fired weapon has a last-shot time of zero, so the timeout comparison passes
immediately and the first shot from a freshly drawn weapon qualifies. That is intended —
drawing a weapon and firing once deliberately is exactly the behaviour being rewarded — but
note that it also means the privilege is not consumed by switching weapons away and back,
since each weapon keeps its own timestamp and neither is reset by being holstered.

The speed comparison is against the limit inclusively at the boundary: a shooter exactly at
the limit still qualifies. Standing still is speed zero, so the common case is unambiguous;
the limit exists to allow a slow walk.

## `make_shot`

**Contract** — stamps the current global clock as the last shot. Called on every shot,
qualifying or not, so any shot restarts the idle period. Does not consult the feature flag,
which is harmless since the timestamp is only read behind it.

**Notes** — that *any* shot resets the timer, not just a qualifying one, is what makes the
rule about holding fire rather than about spacing accurate shots. A player firing a burst
and then a single aimed shot has to wait out the full timeout from the last round of the
burst.
