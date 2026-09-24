# src/xrGame/HUDTarget.cpp

> What the player is looking at: a ray cast from the camera each frame that sees through glass and foliage, and the cursor, name plate and reticle colour derived from whatever it hits.

**Needs** — [`HUDTarget.h`](HUDTarget.h.md) · [`HUDCrosshair.h`](HUDCrosshair.h.md) · [`Level.h`](Level.h.md) · [`entity_alive.h`](entity_alive.h.md) · [`InventoryOwner.h`](InventoryOwner.h.md) · [`inventory_item.h`](inventory_item.h.md) · [`relation_registry.h`](relation_registry.h.md) · [`character_info.h`](../xrServerEntities/character_info.h.md) · [`ai/monsters/poltergeist/poltergeist.h`](ai/monsters/poltergeist/poltergeist.h.md) · [`xrMaterialSystem/GameMtlLib.h`](../xrMaterialSystem/GameMtlLib.h.md) · [`xrCDB/xr_collide_defs.h`](../xrCDB/xr_collide_defs.h.md) · [Seam: Static collision database](../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: one ray query and a screen-space quad per frame; the material lookup is by index into a shared table

## Purpose

Two jobs that share a ray. Each frame a ray is cast along the view axis and the first
thing it meaningfully hits becomes *the target*: its distance drives the depth-of-field
focus and the cursor's screen position, its identity drives the reticle's colour and the
name plate that fades in over it.

The load-bearing decision is what "meaningfully hits" means. The ray does not stop at the
first triangle. Static geometry is tested against its **material's visual transparency
factor**, and the ray keeps going — accumulating a product of those factors — until the
accumulated visibility falls to a third. So a window, a chain-link fence or a bush does
not hide the stalker behind it, but three layers of foliage do. Any *dynamic object* stops
the ray immediately.

## State

```text
RECORD PickResult
  hit       : optional<Object>   # none when the ray ended on static geometry
  range     : real               # distance along the view axis to the accepted hit
  element   : int                # the triangle index when the hit was static geometry
  power     : real               # accumulated visibility, 1 at the camera, product of
                                 # each pierced material's transparency factor
  passes    : int                # how many surfaces were considered; debug readout only

RECORD TargetState
  pick          : PickResult
  info_fade     : real           # 0..1, how far the name plate has faded in
  show_crosshair: bool           # reticle (a weapon is aimed) versus dot cursor
```

Invariants: the accumulated visibility only decreases along the ray; the ray is accepted
as soon as it falls to or below the cutoff, so `power` at the accepted hit is always at
most the cutoff unless the hit was a dynamic object or the first surface. The name plate
is drawn only when the fade has passed its halfway point, and its opacity is the fade
remapped from that half to full — so the plate appears abruptly at half and reaches full
opacity at one.

## `CursorOnFrame`

**Contract** — cast the ray and record the result. Runs once per frame while there is a
controlled entity. The ray starts at the camera, runs along the view direction, and is
limited to just under the current weather's far plane — so fog defines how far the player
can be said to be looking. Ignores the controlled entity itself. Allocates nothing per
frame; the result buffer is reused.

**Invariants** — the recorded distance is clamped to a small minimum, so a target pressed
against the camera still yields a usable focus distance and a cursor that projects on
screen rather than behind it.

```text
FUNCTION cursor_on_frame()
  IF there is no controlled entity THEN RETURN
  pick.hit = none
  pick.range = current weather far plane * 0.99      # fog is the reach of attention
  pick.power = 1
  ray = (camera position, camera direction, pick.range), both static and dynamic, back faces culled
  cast ray, ignoring the controlled entity, calling accept_surface for each hit in order
  IF anything was accepted THEN clamp pick.range to at least the near limit

FUNCTION accept_surface(hit) -> bool            # true means "keep going"
  passes = passes + 1
  IF hit is a dynamic object THEN
    pick = hit
    RETURN false                                 # any object stops the ray
  material = material of the static triangle hit
  power = power * material.visual_transparency
  IF power > see_through_cutoff THEN RETURN true # transparent enough; look past it
  pick = hit
  RETURN false
```

**Notes** — the cutoff is roughly a third. It is the single number that decides whether
the player's crosshair "sees" a character through a dirty window; nothing in the source
derives it.

The far plane being the weather's rather than the renderer's is deliberate: on a foggy
day the player cannot identify a distant figure, and the HUD agrees with the image.

## `Render`

**Contract** — draw the cursor or the reticle at the target's projected position, tinted
by the target's relationship to the player, and fade a name plate in over it. Does
nothing when the crosshair display flags are all off, or when there is no controlled
entity.

**Invariants** — the cursor's on-screen *size* shrinks with distance, but as a weak
function of the projected depth rather than proportionally, so a distant target still has
a visible, clickable-looking mark. The reticle and the cursor are alternatives, never
both: a drawn weapon shows the reticle, anything else shows the dot.

```text
FUNCTION render()
  IF no crosshair flag is set THEN RETURN
  target_point = camera position + view direction * pick.range
  projected    = full transform applied to target_point      # clip space; y is flipped for screen
  size = base_size / projected.w ^ 0.2                        # weak shrink with depth

  colour = neutral white
  IF the name-plate flag is set THEN colour, plate = identify(pick.hit)
  IF the distance-readout flag is set THEN draw the range above the cursor

  IF a weapon is aimed THEN
    reticle.colour = colour ; reticle.render()
  ELSE
    draw a screen-aligned quad of the cursor material at projected, tinted colour
```

## Target identification and the name plate

**Contract** — decide the tint and the text from what the ray hit. Single player and
multiplayer answer differently, and the difference is the whole point of the branch.

**Single player**: a living monster is always hostile. A living character is tinted by
the **relation registry's** verdict on how *it* regards the player — enemy, neutral or
friend — and its plate carries the character's name and its faction. An inventory item
within about two metres shows its item name. Nothing else shows a plate.

**Multiplayer**: a living entity is tinted by team, and the plate carries its object
name. The fade-in speed depends on distance: close targets resolve almost instantly,
distant ones take up to twenty times longer, interpolated linearly between a near and a
far bound; beyond the far bound the plate never appears.

**Invariants** — the fade rises while a plate-worthy target is held and falls twenty
times faster when it is lost, which is what makes the plate snap away when the player
looks off a character but drift in when they look at one. It is clamped to the unit
interval every frame.

**Notes** — the multiplayer branch's distance-dependent reveal is a *reconnaissance*
mechanic: identifying a distant player takes time, and looking away resets it. That is
the only place in the HUD where information is deliberately withheld from the player as
a gameplay rule rather than as a presentation choice.

The poltergeist is special-cased: it is identified even while invisible, because it is
never visible and would otherwise be unidentifiable at all. A rebuild should read this as
"visibility is not the right test for whether a target can be named" and give the
creature an explicit capability instead of naming its class here.

The multiplayer branch contains a contradiction preserved from the original: it is
entered only when the game type is *not* single player, but its body then requires the
game type to *be* single player before drawing anything. The multiplayer name plate is
therefore dead code as shipped. It is documented because a rebuild reading only the
source would faithfully reproduce a feature that never runs.

## `net_Relcase`

**Contract** — when any object is about to be destroyed, drop it from the pick result and
clear the ray-result buffer. Without this the HUD would hold a reference to a destroyed
entity for up to one frame, which the preface's runtime invariants forbid.

## `Load`, `ShowCrosshair`, `GetRQ`, `GetRQVis`, `GetHUDCrosshair`

**Contract** — load the reticle's configuration; switch between reticle and cursor; and
hand out the last pick result, its accumulated visibility, and the reticle itself. The
pick result is read by the depth-of-field effector, which focuses on whatever the player
is looking at.

## Notes

The tint colours, the base cursor size, the near clamp, the two fade speeds and the four
reconnaissance bounds are all fixed constants in this file rather than configuration.
They are tuning, and none of them is derived from anything.
