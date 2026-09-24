# src/xrGame/Actor_Feel.cpp

> What the player notices: which loose items are near and in view, which of them can be picked up, which primed grenades are worth a warning, and how much noise the player is making.

**Needs** — [`Actor.h`](Actor.h.md) · [`Inventory.h`](Inventory.h.md) · [`CustomZone.h`](CustomZone.h.md) · [`Grenade.h`](Grenade.h.md) · [`ExplosiveRocket.h`](ExplosiveRocket.h.md) · [`Level.h`](Level.h.md) · [`HUDManager.h`](HUDManager.h.md) · [`ui/UIMainIngameWnd.h`](ui/UIMainIngameWnd.h.md) · [`xrMaterialSystem/GameMtlLib.h`](../xrMaterialSystem/GameMtlLib.h.md) · [`xrEngine/CameraBase.h`](../xrEngine/CameraBase.h.md) · [Seam: Static collision database](../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: spatial queries and ray casts against the collision database every frame

## Purpose

The actor implements two of the three *feel* senses — touch and sound — and this is their
handler set, plus everything downstream of touch: the pickup logic and the item name
labels floating over loose objects.

The interesting content is that there are **three different pickup schemes** in this one
file, one per shipped game, and they disagree about what "near" means. A rebuild must pick
one per game, not merge them.

## State

```text
  feel_touch_characters : int    # how many of the touched objects are living creatures
  pickup_mode           : bool   # the use action is held
  snd_noise             : real   # loudest sound this actor produced this frame
  pickup_info_radius    : real   # touch radius used by the oldest pickup scheme
  feel_grenade_time     : real   # a grenade younger than this is not worth warning about
  auto_pickup box and offset     # multiplayer automatic pickup volume
```

**Invariant** — the character counter is maintained by the two touch callbacks in step, so
it is exact as long as every add is matched by a delete. It exists so that code asking
"is somebody standing on me" does not have to re-scan the touch set.

## `feel_touch_contact` — what the actor notices at all

**Contract** — the filter that decides whether an object joins the touch set. Two
categories qualify: a *useful* inventory item that is not already owned by someone, and any
other inventory owner — a creature — that is not this actor. Everything else is ignored.

**Invariants** — filtering at the membership test rather than at every use site is what
keeps the touch set small; a level has thousands of objects and a handful of them are items
lying loose.

## `feel_touch_on_contact` — the anomaly exception

**Contract** — a second filter applied to an object already inside the touch radius. Only
anomalies are treated specially: an anomaly counts as touched only when the actor's
*centre*, as a small sphere, is genuinely inside the anomaly's own volume. Everything else
passes.

**Notes** — anomalies are large and irregularly shaped, and a radius test against their
origin would trigger them from outside. Their own containment test is the authority.

## `feel_touch_new` · `feel_touch_delete`

**Contract** — maintain the living-creature count as objects enter and leave the touch set.

## `feel_sound_new`

**Contract** — the sound sense, used only to measure the actor's *own* noise: a sound event
whose emitter is this actor raises the frame's noise level to the loudest such sound.
Sounds made by others are ignored here — the player hears them through the audio device,
and the AI's hearing is elsewhere.

**Notes** — this is a maximum, not a sum. Two simultaneous footsteps are as loud as one.
The value feeds the creature perception system's model of how detectable the player is.

## `CanPickItem`

**Contract** — is this item both in view and unobstructed? Answers false when the item is
invisible, and otherwise: skip the whole test for anything closer than a quarter of a
metre (it is on top of the player); reject anything outside the view frustum; and cast a
ray from the camera to the item's centre, rejecting it if anything blocks.

**Invariants** — the blocking test has two exemptions and both matter. A dynamic object
blocks only if it is flagged as an *obstacle*, so a fellow creature standing between the
player and a rifle does not hide it. Static geometry blocks only if its material is not
flagged *passable*, so foliage and chain-link do not hide it either. The actor's own body
never blocks.

**Notes** — the ray stops at the item's centre rather than passing through, and the item
itself is excluded from the query. The quarter-metre short-circuit exists because a ray of
near-zero length has no well-defined direction.

## `PickupModeUpdate` — the oldest scheme

**Contract** — touch-radius pickup. While the use action is held, if the object the actor
is currently looking at is a useful inventory item that is not script-gated and is not
under a touch denial, use it and tell the game mode a pickup happened. Then refresh the
touch set at the label radius and draw a name label over every touched item that passes the
visibility test.

**Notes** — single player only. "The object we are looking at" is maintained by the
actor's own look ray elsewhere; this scheme relies on it entirely and the radius only
controls the labels.

## `PickupModeUpdate_COD` — the newest scheme

**Contract** — nearest-to-the-aim-line pickup with a persistent on-screen prompt. Runs only
for the viewed entity, and clears the prompt entirely unless the actor is alive and in
first person with the feature enabled.

```text
FUNCTION pickup_update()
  candidates = spatial query for collideable objects inside the view frustum
  best = none; best_offset = +infinity

  FOR EACH candidate that is an inventory item
    SKIP if it already has an owner, cannot be taken, is a rocket in flight,
         or is a thrown item that is not yet useful (a live grenade)
    SKIP if its centre is more than 2 metres from the actor        # squared test
    # The ranking is distance from the AIM LINE, not from the actor:
    offset = |centre - projection of centre onto the camera ray|
    SKIP if offset > 1 metre
    keep the smallest offset among non-destroyed candidates

  IF a best candidate exists THEN
    drop it if it fails the line-of-sight test, is under a touch denial,
      or is invisible

  show it in the pickup prompt (or clear the prompt)
  IF the use action is held AND a candidate remains THEN
    use it, tell the game mode, and end pickup mode unless
    multi-item pickup is configured
```

**Invariants** — ranking by perpendicular distance from the aim line rather than by
distance from the player is the whole difference between this scheme and the older one, and
it is what makes the prompt feel like it is tracking the crosshair. Both thresholds — two
metres of reach, one metre of aim tolerance — are compared as squared values, so a rebuild
must square its own thresholds or change the reach.

**Notes** — live grenades and rockets in flight are excluded by name. Picking up an armed
grenade would be a very specific kind of bug.

The frustum is built twice, identically, in the same call. Harmless duplication.

## `Check_for_AutoPickUp` — the multiplayer scheme

**Contract** — walking over an item picks it up, with no aiming and no prompt. Multiplayer
only, for the controlled living actor, and only when the setting is on. A box offset from
the actor's position is queried for collideable items; every takeable, non-denied,
non-grenade item whose position falls inside the box is picked up.

**Invariants** — in the two deathmatch modes, a weapon is skipped when its natural slot is
already occupied, so running over a rifle does not replace the one you are carrying.
Grenades are excluded unconditionally, which differs from the other two schemes only in
that they exclude *live* grenades.

**Notes** — the box test is done twice: once as the spatial query and once as a point
containment test against the same box. The second is what makes the volume exact, since the
spatial query is conservative.

## `PickupInfoDraw`

**Contract** — draws an item's name at its projected screen position. Projects the item's
origin through the full view-projection transform, rejects anything behind the camera or
outside normalized device coordinates, converts to pixels and draws centred.

**Notes** — the rejection uses both the depth and the homogeneous coordinate, which is the
standard guard against the sign flip an object behind the camera produces. The font is
named explicitly; the localization set it belongs to is part of the shipped data.

## `Feel_Grenade_Update`

**Contract** — finds primed grenades near the actor and hands them to the interface so it
can draw a warning marker. Single player only. A grenade qualifies when it is not being
destroyed, was not thrown by this actor, is *not* still useful (meaning it is armed rather
than lying as an item), and has been in flight for longer than a configured time.

**Invariants** — the flight-time threshold is what stops the marker appearing on a grenade
the instant it leaves a thrower's hand, which would make the warning a perfect early alarm
rather than a fair one.

**Notes** — the interface owns the marker set and is told the actor's position each update
so it can place the markers; this file only decides membership.
