# src/xrGame/CarScript.cpp

> The vehicle as scripts see it: a turret to aim and fire, an engine to start and stop, a fuel tank to read and write, and a way to blow the whole thing up.

**Needs** — [`Car.h`](Car.h.md) · [`CarWeapon.h`](CarWeapon.h.md) · [`script_game_object.h`](script_game_object.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: one declarative registration block

## Purpose

Exports the vehicle class to the script layer. Frozen by conformance criterion 10: every
name below ships in the games' own scripts or in the modifications the project exists to
keep working, so a rebuild may reimplement any of them and may rename none.

The class is declared as deriving from **both** the game-object facade and the holder
interface, which is what lets a script treat one handle as "a thing in the world" and "a
thing the actor can be inside" without asking which.

## State

`Stateless.`

## `script_register`

**Contract** — register the vehicle class, one enumeration and its method set with the
script virtual machine. Runs once at startup, from the game module's registration sweep.

Exported as `CCar`:

- **`wpn_action`** — the mounted weapon's command vocabulary, exposed as an enumeration on
  the car rather than on the weapon, because the weapon is not itself a script class:
  desired direction, desired position, activate, fire, auto-fire, return to default
  direction.
- **`Action`, `SetParam`, `CanHit`, `FireDirDiff`, `IsObjectVisible`, `HasWeapon`** — the
  turret. `SetParam` is exported in its three-component form only; the two-component
  overload is commented out, so a script cannot set a screen-space aim point.
- **`CurrentVel`** — the vehicle's linear velocity.
- **`GetfHealth`, `SetfHealth`, `ChangefHealth`** — health. The first two are re-exports of
  the base entity's, present because the script layer resolves overloads by declared type
  and the base's are not visible through this class.
- **`SetExplodeTime`, `ExplodeTime`, `CarExplode`** — the delayed-action fuse, in
  milliseconds, and the immediate destruction.
- **`get_fuel` / `set_fuel`, `get_fuel_tank` / `set_fuel_tank`, `get_fuel_consumption` /
  `set_fuel_consumption`** — the fuel model.
- **`GetfFuel`, `SetfFuel`, `GetfFuelTank`, `SetfFuelTank`, `GetfFuelConsumption`,
  `SetfFuelConsumption`, `ChangefFuel`** — **the same six operations again under different
  names**, plus a relative fuel change.
- **`PlayDamageParticles`, `StopDamageParticles`** — force the damage smoke on or off.
- **`StartEngine`, `StopEngine`, `IsActiveEngine`, `HandBreak`, `ReleaseHandBreak`,
  `GetRPM`, `SetRPM`** — drive the vehicle without a driver.

**Notes** — the duplicate fuel surface is not a mistake to clean up. Two modification
communities added the same feature under two naming conventions and both sets of names are
now in shipped scripts, so **both must exist**. This is what criterion 10 costs in
practice, and it is the clearest example of it in the game layer: a rebuild that exports
only the tidy set breaks scripts.

`SetRPM` writes the engine's smoothed speed directly, which the next physics step will
immediately smooth back toward what the wheels imply. It is useful only for a stopped
vehicle, and a script using it to "rev" a car is fighting the engine model.
