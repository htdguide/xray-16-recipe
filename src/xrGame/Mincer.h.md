# src/xrGame/Mincer.h

> Declares the meat-grinder anomaly implemented in [`Mincer.cpp`](Mincer.cpp.md).

**Needs** — [`GraviZone.h`](GraviZone.h.md) · [`TeleWhirlwind.h`](TeleWhirlwind.h.md) · [`PhysicsShellHolder.h`](PhysicsShellHolder.h.md) · [`PHDestroyable.h`](PHDestroyable.h.md) · [`Mincer.cpp`](Mincer.cpp.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`Mincer.cpp`](Mincer.cpp.md) · [`mincer_script.cpp`](mincer_script.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CMincer`, the anomaly that suspends everything inside it and tears it apart on
discharge. Substance is in [`Mincer.cpp`](Mincer.cpp.md).

The declaration's own content is that the class carries **two roles at once**: it is a gravity
zone, and it is a listener for destructible-object notifications. The second role is how the
zone learns that something it was holding has come apart, so it can play the tearing effects
and deliver the scattering impulse. A rebuild must provide both attachment points.

Exported units:

- `CMincer` — the anomaly, owning a telekinesis controller, the torn-body particle name, the
  tearing sound and the player-specific blowout radius fraction.
- `Telekinesis` — hands out the controller, so the telekinesis machinery can reach it.
- `Load` / `net_Spawn` / `net_Destroy` — the lifecycle; spawn places the controller, destroy
  releases every hold.
- `OnStateSwitch` — grabs everything on entering blowout, releases on leaving.
- `BlowoutState` — the once-only destructive release at the discharge instant.
- `feel_touch_contact` / `feel_touch_new` — narrow the zone's interest to objects with a
  physics body, and grab a late arrival mid-blowout.
- `AffectPullAlife` / `AffectPullDead` / `AffectThrow` — the per-tick forces; the dead case is
  deliberately empty.
- `NotificateDestroy` — the destructible-object callback that plays the tearing effects and
  delivers the impulse.
- `Center` / `ThrowInCenter` — the zone centre and the whirlwind axis, which differ in height.
- `BlowoutRadiusPercent` — a separate lethal radius fraction for the player.

The class registers itself with the script binding layer as a subclass of the game object
facade.
