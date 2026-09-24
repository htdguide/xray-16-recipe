# src/xrGame/Actor_Flags.h

> The player-behaviour switch set, plus the mouse-sensitivity ranges and the sleep duration — every one of them a console-settable global.

**Needs** — _(none beyond the shared prelude)_
**Used by** — [`Actor.cpp`](Actor.cpp.md) · [`Actor.h`](Actor.h.md) · [`Actor_Network.cpp`](Actor_Network.cpp.md) · [`Spectator.h`](Spectator.h.md) · [`Weapon.h`](Weapon.h.md) · [`console_commands.cpp`](console_commands.cpp.md) · [`UISleepStatic.cpp`](ui/UISleepStatic.cpp.md)
**Tier floor** — T3: a named bit set and five tuning values.

## Purpose

One flag word holds every switch that changes how the player's entity behaves, and it is
reached by name from a dozen files. The names are frozen: each one is bound to a console
variable, so shipped configuration files, the user's settings file and modification scripts
address them, and conformance criterion 3 requires the whole set to be accepted.

## State

```text
ENUM ActorFlag                # bit positions in one 32-bit word; gaps are retired flags
  god_mode                = bit 0    # no damage at all
  no_clip                 = bit 1    # free flight, controller position not read back
  unlimited_ammo          = bit 3    # bit 2 is retired and must stay unused
  run_backward            = bit 4    # backward movement may use the run multiplier
  auto_pickup             = bit 5    # walking over an item takes it
  third_person            = bit 6    # start in the third-person camera, set from a
                                     #   command-line switch rather than the console
  dynamic_music           = bit 7
  god_mode_realtime       = bit 8    # damage applies but health is restored at once
  important_save          = bit 9    # mark autosaves as not to be overwritten
  crouch_toggle           = bit 10   # crouch latches instead of being held
  multi_item_pickup       = bit 11   # hold use to sweep up everything in reach
  loading_stages          = bit 12
  always_use_attitude_sensors = bit 13  # gamepad tilt steers always, not only when aiming
  use_tracers             = bit 14
```

**Invariants** — bit 2 is *absent*, not zero: a retired flag's position is never reused,
because a settings file written by an older build still carries its value and reusing the
position would silently turn on an unrelated behaviour.

## `psActorFlags`

**Contract** — the live flag word. Its startup value is itself a decision — real-time god
mode, auto-pickup, backward running, important-save marking, multi-item pickup and tracers
are on; plain god mode, no-clip and unlimited ammo are off — and that is what an install
with no settings file plays like. Defined in [`Actor.cpp`](Actor.cpp.md).

## `GodMode`

**Contract** — answers whether damage should be ignored entirely. Separate from a direct
flag test because two flags express invulnerability differently: one suppresses the hit,
the other lets it land and restores the health immediately, and only the first one belongs
here.

## Look and cursor sensitivity

**Contract** — two triples: a minimum, a maximum and a step, for the free-look sensitivity
and for the menu cursor's. The step is what one press of the sensitivity binding moves; the
bounds clamp the result.

**Notes** — the two are separate because they are measured in different things. Look
sensitivity is degrees of turn per unit of mouse travel and its useful range is wide
(15 to 60); cursor sensitivity is pixels per unit and its range is narrow (5 to 15). A
rebuild that collapses them into one setting makes menus unusable at playable look speeds.

## `psActorSleepTime`

**Contract** — how many in-game hours one sleep action advances the clock by. One, by
default.
