# src/xrGame/WeaponCustomPistol.cpp

> A semi-automatic firearm: one round per trigger pull, no burst, and the trigger is not released until the shot's cadence has elapsed.

**Needs** — [`WeaponCustomPistol.h`](WeaponCustomPistol.h.md) · [`WeaponMagazined.h`](WeaponMagazined.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: two overrides of the firing state machine.

## Purpose

Semi-automatic fire is not a burst of length one. A burst of length one would let the
player empty a magazine as fast as the update loop runs; a semi-automatic weapon must
fire once per *pull*, and the next pull may not come before the weapon has cycled. This
file is those two rules, and nothing else.

The name is historical: the class covers the two sniper rifles as well as every pistol
(see [`WeaponSVD.h`](WeaponSVD.h.md), [`WeaponSVU.h`](WeaponSVU.h.md)).

## State

`Stateless.` It only sets fields the magazined weapon owns.

## `switch2_Fire`

**Contract** — entering the fire state arms exactly one round and does **not** set the
"currently firing" flag.

```text
FUNCTION switch_to_fire(weapon)
  weapon.fire_single_shot = true      # one round is guaranteed
  weapon.firing            = false    # but the burst loop's condition is false
  weapon.shots_fired       = 0
  weapon.stopped_after_queue = false
```

The burst loop in [`WeaponMagazined.cpp`](WeaponMagazined.cpp.md) continues while
*(firing OR fire_single_shot)*. With `firing` false and the single-shot latch set, the
loop runs exactly once and then stops, no matter how long the player holds the trigger.
That is the whole mechanism.

**Notes** — the automatic weapon's version of this procedure also validates that the
weapon really is the carrier's active item and, on a network client, restarts firing.
Neither is repeated here, so a semi-automatic weapon on a client does not re-trigger
itself from a replicated state — which is correct, because each of its shots is its own
authoritative event.

## `FireEnd`

**Contract** — the trigger release is *deferred* until the weapon has cycled.

```text
FUNCTION fire_end(weapon)
  IF the shot clock has expired THEN
    clear pending
    base.fire_end()
  # otherwise: ignore the release entirely; the player must pull again
```

**Invariants** — this is what enforces the rate of fire on a semi-automatic weapon.
Releasing the trigger early leaves the weapon pending, which makes every input handler in
the hierarchy refuse the next pull until the clock runs down and a later release clears
it. A rebuild that simply rate-limits `FireStart` gets a subtly different feel: the
original's weapon is *unresponsive* between shots, not merely rate-limited, and the
difference shows in whether a queued click fires late or not at all.

## `GetCurrentFireMode`

**Contract** — always one. A semi-automatic weapon has no selector even if its
configuration section lists fire modes.
