# src/xrGame/CarLights.cpp

> A vehicle headlight: a spot light, a glow sprite and a bone that is only drawn while the light is on, kept together so that switching one switches all three.

**Needs** — [`CarLights.h`](CarLights.h.md) · [`Car.h`](Car.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [`xrPhysics/IPHWorld.h`](../xrPhysics/IPHWorld.h.md) · [`xrEngine/Render.h`](../xrEngine/Render.h.md) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — reached through its declarations in [`CarLights.h`](CarLights.h.md); callers name that, not this file.
**Tier floor** — T2: holds two renderer resources and composes one transform per frame

## Purpose

A headlight in this engine is three things that must agree: a shadow-casting spot light, a
glow sprite so the lamp is visible from outside its own cone, and a piece of the model —
the lit lens — that is hidden while the lamp is off. Keeping them in one record is the
whole design, and the invariant that they are never out of step is asserted at runtime.

The lights are read from a per-vehicle configuration list, so a model with no `lights`
section simply has none.

## State

```text
RECORD CarLight
  light   : spot light handle      # shadow-casting
  glow    : glow sprite handle
  bone_id : bone                   # the lens bone, visible only while lit
  holder  : CarLights

RECORD CarLights
  lights : list<CarLight>
  car    : vehicle
```

Invariants:

- the spot light and the glow are **always in the same active state**. The query for "is
  this light on" asserts it before answering.
- the lens bone's visibility mirrors that state.
- none of these operations may run inside a physics step, because they touch the renderer
  and force a pose recomputation.

## `SCarLight::ParseDefinitions`

**Contract** — create the two renderer resources from one configuration section and place
them on a named bone. Leaves the light off and the lens bone hidden.

```text
FUNCTION parse(section)
  light = renderer.create_light(); type = spot; casts shadow
  glow  = renderer.create_glow()
  colour = section.color; apply it to both
  light.range, light.cone (authored in degrees), light.texture  from the section
  glow.texture, glow.radius from the section
  bone_id = the bone the section names
  set both inactive; hide the lens bone, with its children
```

**Notes** — the colour is shared between the light and the glow deliberately: an artist
tunes one value and the lamp and the light it casts match. A commented-out line would have
scaled the light's colour by a brightness taken from the player's torch settings; it is
disabled, so vehicle lights ignore torch brightness.

Each light's section is named independently in the vehicle's `lights` / `headlights` list,
so two lamps on one car may differ in colour, range and cone.

## `TurnOn`, `TurnOff`, `Switch`, `isOn`

**Contract** — switch all three pieces together. Turning on additionally **forces a pose
recomputation**, because the lens bone was hidden and its transform is stale; the light is
positioned from that transform in the same call.

## `SCarLight::Update`

**Contract** — place the light and glow at the lens bone's current world pose. Skipped
entirely while off.

```text
FUNCTION update()
  IF off THEN RETURN
  pose = vehicle transform composed with the lens bone's transform
  light.rotation  = (pose forward, pose right)
  glow.direction  = pose forward
  glow.position   = light.position = pose translation
```

**Notes** — the light follows the *bone*, not the vehicle, so a headlight mounted on a
part that moves — or a lamp on a door — swings with it.

## `CCarLights` — the collection

**Contract** — `ParseDefinitions` builds one light per name in the vehicle's list;
`Update`, `SwitchHeadLights`, `TurnOnHeadLights` and `TurnOffHeadLights` fan out to all of
them; the destructor releases them.

**Notes** — the vehicle has exactly one headlight group, switched as a unit. The commented-out
fields in the declaration show what was intended and never built: separate near and far
beams, indicators, brake lights, sidelights and door lights, each as a range into the same
list. A rebuild wanting those has the shape to follow, and no data to feed it — the shipped
models declare only `headlights`.

## `findLight`, `IsLight`

**Contract** — find a light by the bone it sits on. Returns whether one was found, and by
reference the light.

**Notes** — **this function reads past the end of its list when the bone is not found.**
It dereferences the search result before comparing it against the end. `IsLight` calls it
with no other purpose than the boolean, so any query for a bone that is not a light is
undefined. A rebuild must order the two operations the other way; there is nothing to
preserve here but the intent.
