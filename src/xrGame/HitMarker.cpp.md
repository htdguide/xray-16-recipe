# src/xrGame/HitMarker.cpp

> The directional damage indicator and the grenade warning: full-screen sprites rotated to point at where the hit came from, or at where a live grenade is, fading out on a shared authored curve.

**Needs** — [`HitMarker.h`](HitMarker.h.md) · [`Grenade.h`](Grenade.h.md) · [`xrEngine/LightAnimLibrary.h`](../xrEngine/LightAnimLibrary.h.md) · [`xrUICore/Static/UIStaticItem.h`](../xrUICore/Static/UIStaticItem.h.md) · [`Include/xrRender/UIShader.h`](../Include/xrRender/UIShader.h.md) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: two expiring queues and one rotated sprite each

## Purpose

Two indicators that answer the same question — *which way?* — and are built the same way,
which is why they share a file.

The mechanism is worth stating because it is not the obvious one: each marker is a
**centred, full-screen-width sprite rotated about the screen centre**. The texture is
authored so that its meaningful part points one way; rotating the whole sprite by the
bearing difference between the camera and the threat makes it point at the threat. There
is no arrow placed on a circle, no screen-edge clamping and no per-marker geometry.

The fade is likewise not hand-written: both marker kinds sample a **named light
animation** — a shared authored colour-over-time curve — and take its alpha. The curve's
own length is the marker's lifetime, so an artist changes how long a hit indicator lingers
by editing the animation, not the code.

## State

```text
RECORD HitMark
  sprite      : Sprite   # centred, half the base screen width square
  start_time  : real     # when the hit landed
  bearing     : real     # world heading the hit came from
  curve       : LightAnimation   # shared, named; its length is this marker's lifetime

RECORD GrenadeMark
  grenade     : reference to a Grenade
  detached    : bool     # the grenade is gone or exploding; stop tracking it
  sprite      : Sprite   # centred, a fixed 640-unit square in base UI coordinates
  last_update : real     # when the bearing was last refreshed
  bearing     : real     # world heading toward the grenade
  curve       : LightAnimation   # the same shared curve, sampled at double rate

RECORD HitMarker
  hit_marks     : queue<HitMark>      # oldest first
  grenade_marks : queue<GrenadeMark>  # oldest first
```

Invariants:

- Both collections are **queues expiring in order**: because every marker of a kind has
  the same lifetime and they are appended in time order, the front is always the first to
  expire, and retirement is a loop from the front rather than a scan. A rebuild that
  gives markers differing lifetimes loses this and must scan.
- A grenade marker outlives its grenade. Once detached it keeps fading with its last
  bearing rather than disappearing, so a grenade that explodes does not make its warning
  vanish at the instant it matters most.
- At most one marker per grenade exists at a time.

## `CHitMarker` construction and `InitShader` / `InitShader_Grenade`

**Contract** — both textures are named in one configuration section; the hit mark's is
required and the grenade mark's is optional. Both are bound to the same generic
heads-up material with the texture swapped, and both can be re-bound at runtime, which is
how a modified game changes the indicator art without restarting.

## `Hit`

**Contract** — record a hit from a world direction. Appends a marker whose bearing is the
**reverse** of the supplied direction.

**Invariants** — the reversal is the whole semantic: the caller passes the direction the
damage was *travelling*, and the player needs to look the way it came *from*.

## `Render`

**Contract** — retire expired markers from the front of both queues, then draw every
survivor of both kinds, each rotated by the camera's heading negated plus its own bearing.

**Invariants** — the camera's heading is negated because the marker rotates in screen
space while the bearing is in world space; adding the marker's world bearing to the
negated camera bearing yields the bearing relative to where the player is facing, which is
exactly the screen rotation wanted.

```text
FUNCTION render()
  camera_heading = heading of the camera direction
  WHILE the oldest hit mark has expired: drop it
  WHILE the oldest grenade mark has expired: drop it
  FOR EACH marker IN both queues
    alpha = marker.curve sampled at its own elapsed time
    draw the sprite rotated by (-camera_heading + marker.bearing), at that alpha
```

## `AddGrenade_ForMark`

**Contract** — begin warning about a grenade, unless a live marker already tracks it.
Reports whether it added one, so the caller knows whether this is a new threat.

**Invariants** — the duplicate check skips detached markers, so a second grenade with the
same entity identifier — which happens, since identifiers are reused — can be tracked
again after the first has been let go.

## `Update_GrenadeView`

**Contract** — once per frame, for every still-attached grenade marker: if the grenade has
begun exploding, detach; otherwise recompute the bearing from the player toward the
grenade and refresh the marker's timer.

**Invariants** — refreshing the timer is what makes the grenade warning *persistent* while
the hit indicator is *transient*. A hit marker's clock runs from the hit; a grenade
marker's clock runs from its last update, so it never fades while the grenade is still
there, and begins fading the moment tracking stops.

```text
FUNCTION update_grenade_view(player_position)
  FOR EACH marker IN grenade_marks
    IF detached THEN CONTINUE
    IF marker.grenade is exploding THEN detached = true ; CONTINUE
    marker.bearing = heading from player_position to the grenade's centre
    marker.last_update = now                        # resets the fade
```

## `net_Relcase`

**Contract** — when any object is destroyed, detach every grenade marker that references
it. Does not stop at the first match, deliberately: nothing guarantees uniqueness once a
marker has been detached and an identifier reused.

**Invariants** — this is the only thing keeping a grenade marker from holding a destroyed
object. A detached marker never dereferences its grenade again.

## `SHitMark`

**Contract** — a marker covering half the base screen width, square, centred, bound to the
shared curve named for the hit indicator. It is active while its elapsed time is within
the curve's length, and it draws at the curve's alpha at that time.

## `SGrenadeMark`

**Contract** — the same, at a fixed size in base interface units, and with the curve
sampled at **twice** the elapsed rate — so the grenade warning fades out in half the time
a hit indicator does, once it stops being refreshed.

**Notes** — the doubling is the only difference in behaviour between the two marker kinds,
and it is applied to both the activity test and the colour sample so the two stay
consistent. Reusing the hit indicator's curve rather than authoring a second one is why
the doubling exists at all; a rebuild with a second named curve has neither.

The grenade sprite's size is a literal in base interface coordinates while the hit
sprite's is derived from the base width. The two are inconsistent and neither is
explained; the grenade sprite is simply large enough to fill the screen at the base
resolution.
