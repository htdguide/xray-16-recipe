# src/xrGame/WeaponStatMgunIR.cpp

> The mounted gun's input: every pointing device funnelled into one procedure that moves the desired direction, never the barrel.

**Needs** — [`WeaponStatMgun.h`](WeaponStatMgun.h.md) · [Seam: Windowing and input](../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: scaling and sign flips over input deltas.

## Purpose

While the actor is mounted, the gun is an input receiver. This file is the whole of that
role, and its content is one decision: **input moves the desired direction, and only the
desired direction.** The barrel chases it with inertia and its own joint limits, in
[`WeaponStatMgun.cpp`](WeaponStatMgun.cpp.md). Nothing here can point the gun somewhere
the mount cannot reach; the clamp lives with the solve, not with the input.

The separate file exists because the same three-line body is reached from four device
paths with four different sensitivity sources.

## State

`Stateless.`

## `OnAxisMove` — the common path

**Contract** — the only procedure that writes the desired direction. Pure apart from that
one write.

```text
FUNCTION on_axis_move(gun, dx, dy, scale_x, scale_y, invert_x, invert_y)
  RETURN IF dx and dy are both zero
  yaw, pitch = the heading and pitch of gun.desired_direction
  yaw   = yaw   - (invert_x ? -1 : +1) * dx * scale_x
  pitch = pitch - (invert_y ? -1 : +1) * dy * scale_y * 0.75
  gun.desired_direction = direction from (yaw, pitch)
```

**Invariants** — vertical movement is scaled by **three quarters** relative to
horizontal, on every device. This is a feel decision for a heavy mount, not an aspect
correction, and it is applied after the player's own sensitivity.

Both deltas are *subtracted*, so the device's positive axis and the world's yaw run
opposite ways. That sign is the engine's convention, shared with the free camera.

## Device paths

**Contract** — four callers, each supplying sensitivity and inversion from the player's
settings, all refusing to act for a remotely controlled gun.

| Device | Scale | Inversion |
|---|---|---|
| mouse | mouse sensitivity × scale ÷ 50, same on both axes | vertical only, from the mouse-invert setting |
| controller stick (press and hold) | separate horizontal and vertical stick sensitivities × scale | both axes, from the controller settings |
| controller motion sensor | sensor sensitivity ÷ 50, same on both axes | both axes, from the controller settings |

The mouse divisor of 50 and the sensor's are the arbitrary normalizations that make the
settings' numbers land in a usable range; they carry no other meaning.

**Notes** — the horizontal mouse-inversion setting is not consulted at all: only the
vertical flag is read. Inverting the horizontal axis therefore has no effect on a mounted
gun while it does on the free camera.

## Firing and the controller's look binding

**Contract** — the fire binding starts firing on press and ends it on release. The
controller's look binding is routed to the axis move on both press and hold and ignored
on release; every other controller command falls through to the keyboard handlers, so one
set of bindings serves both devices.

The "key held" hook is empty: the gun's firing is edge-triggered, and aiming comes from
axis events rather than from held keys.

**Invariants** — every entry point refuses for a remotely controlled gun. A remote gun's
direction and firing state arrive through replication, and letting local input touch
either would fight the network.
