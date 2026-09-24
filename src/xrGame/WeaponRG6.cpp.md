# src/xrGame/WeaponRG6.cpp

> The revolving grenade launcher: a shotgun's shell-at-a-time reload feeding a launcher that spawns a real, physically simulated grenade for every round loaded.

**Needs** — [`WeaponRG6.h`](WeaponRG6.h.md) · [`WeaponShotgun.h`](WeaponShotgun.h.md) · [`RocketLauncher.h`](RocketLauncher.h.md) · [`ExplosiveRocket.h`](ExplosiveRocket.h.md) · [`Level.h`](Level.h.md) · [`Actor.h`](Actor.h.md) · [`xrPhysics/MathUtils.h`](../xrPhysics/MathUtils.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: a ballistic solve and a ray query per trigger pull.

## Purpose

Most weapons fire a *bullet* — an abstraction the bullet manager traces and discards.
This one fires an **object**: a grenade with its own server record, its own physics, its
own fuse and its own explosion. That difference is what the file is about.

The consequence is that a loaded round is not only a cartridge in a magazine but also a
spawned, attached grenade held by the launcher. The two must be kept in step: every
shell pushed into the cylinder spawns a grenade, and every shot detaches one. The file's
whole job is maintaining that correspondence across spawn, reload and fire.

The second interesting piece is the **aim-assist solve**: an actor firing a scoped
grenade launcher gets a lobbed trajectory computed to land where the crosshair points,
rather than a flat throw along the sight line.

## State

`Stateless` of its own — it holds the shotgun's reload state and the rocket launcher's
attached-rocket list, and adds nothing.

**Invariant** — the number of attached grenades equals the number of rounds in the
magazine. Every path that changes one changes the other: spawn fills the gap, the reload
override spawns one per shell it fails to account for, and firing detaches one.

## `net_Spawn` — restoring the grenades

**Contract** — after the shotgun half spawns, any rounds the server record said were
loaded have no grenades behind them, so they are created.

```text
FUNCTION spawn_from_record(launcher, record)
  RETURN IF the shotgun half's spawn failed
  IF ammo_elapsed > 0 AND no grenade is attached THEN
    fake_name = the FIRST ammo type's "fake_grenade_name"
    IF fake_name is non-empty THEN
      spawn and attach one grenade of that name per round in the magazine
```

**Invariants** — the grenade class is looked up from `ammo_types[0]`, not from the
magazine's actual contents. A launcher restored with a non-default round type therefore
gets the default grenade. In practice these weapons have one ammo type.

**Notes** — the grenade is called *fake* because it is not the item the player carries;
it is a purpose-built projectile class named by the ammunition section. The indirection
is what lets one round section describe both an inventory item and a flying object.

## `FireStart` — the launch

**Contract** — refuses unless the weapon is idle and at least one grenade is attached.
Everything after the refusal happens in one call: there is no burst loop, no shot clock
and no `FireTrace` — this weapon bypasses the bullet path entirely.

```text
FUNCTION fire_start(launcher)
  RETURN unless the state is idle AND a grenade is attached
  shotgun_half.fire_start()                  # runs the normal firing bookkeeping

  position, direction = launcher.muzzle, launcher.aim_direction
  IF the carrier is an entity THEN
    ask the carrier for its aim: position, direction = carrier.fire_params(launcher)

  launch_frame = a basis with forward = direction, origin = position

  # --- aim assist, single player only, only while scoped, only for the actor -----
  IF single player AND scoped AND the carrier is the actor THEN
    disable collision on the carrier and on the weapon
    hit = ray query from position along direction, 300 m, against static geometry
    re-enable collision on both
    IF hit THEN
      displacement = direction * hit.range         # where the crosshair points
      solutions = throw directions that carry `displacement` at the launcher's
                  muzzle speed under the effective gravity     # 0, 1 or 2 of them
      IF any solution exists THEN direction = the first (the flatter arc)

  direction = normalize(direction) * launcher.muzzle_speed
  launch the attached grenade from launch_frame with that velocity
  record the carrier as the grenade's initiator          # for kill attribution
  IF we are authoritative THEN broadcast a launch-rocket event naming the grenade
  detach the grenade                                     # it is the world's now
```

**Invariants** —

- the collision on the carrier and the weapon is disabled around the ray query and
  restored immediately, because the query must not hit the shooter's own body or the
  weapon in his hands. The restore must happen on every path out;
- the ray is 300 metres and hits **static geometry only**, so the assist aims at walls
  and terrain, not at creatures. Firing at a stalker in the open leaves the throw flat;
- the ballistic solve returns up to two arcs (a flat one and a lobbed one) and the flat
  one is always chosen;
- the grenade is detached *after* the launch, so a failure between the two would leave a
  flying grenade the launcher still believes it holds.

**Notes** — the aim assist is single-player and scoped only, so the multiplayer weapon
throws flat. That asymmetry is deliberate: the assist is a large accuracy gift.

The "no active item" case is logged and then proceeds anyway, which is a diagnostic left
in place rather than a guard.

## `AddCartridge` — keeping grenades in step with shells

**Contract** — wraps the shotgun's shell push and spawns one grenade per shell that was
actually taken.

```text
FUNCTION add_cartridge(launcher, cnt) -> int
  remaining = shotgun_half.add_cartridge(cnt)     # returns what it could NOT push
  taken     = cnt - remaining
  fake_name = the CURRENT ammo type's "fake_grenade_name"
  spawn and attach `taken` grenades of that name
  RETURN remaining
```

**Notes** — the source's loop counts down the local `taken` and then returns it, which is
always zero. The intended return is the shotgun's `remaining`, and returning zero means
the reload sub-state machine's "could not take one" signal never fires from this weapon —
the sequence instead stops on the magazine-full guard. The observable behaviour is the
same for a launcher, whose cylinder is small and whose ammunition is scarce, which is
why the bug survives. A rebuild should return the shotgun's figure.

## `OnEvent` — grenade ownership

**Contract** — three authoritative events keep the attached-grenade list correct:

| Event | Effect |
|---|---|
| ownership taken | attach the named grenade to this launcher |
| ownership rejected | detach it without launching |
| rocket launched | detach it, marked as launched |

The launcher's own half of the handler runs *after* the shotgun's, so the state change
and ammunition bookkeeping have already been applied when the grenade list is adjusted.

## `Load`

**Contract** — loads both halves: the rocket launcher's parameters (muzzle speed,
gravity behaviour) and the shotgun's (magazine, sounds, tri-state reload). Order is
launcher first, shotgun second; nothing depends on it.

**Notes** — this class inherits from two bases, a launcher and a shotgun, which is why
every entry point names which half it is calling. The decision underneath is composition:
a grenade launcher *has* a projectile-spawning capability and *has* a shell-fed magazine.
A rebuild should express it that way and drop the qualification.
