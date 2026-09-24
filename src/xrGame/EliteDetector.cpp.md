# src/xrGame/EliteDetector.cpp

> The two detector models with a real screen: a rotating plan view of nearby artefacts drawn onto a bone of the device's own model, and the scientific variant that also shows anomalies.

**Needs** — [`EliteDetector.h`](EliteDetector.h.md) · [`CustomDetector.h`](CustomDetector.h.md) · [`ui/ArtefactDetectorUI.h`](ui/ArtefactDetectorUI.h.md) · [`player_hud.h`](player_hud.h.md) · [`Include/xrRender/UIRender.h`](../Include/xrRender/UIRender.h.md) · [`ui/UIXmlInit.h`](ui/UIXmlInit.h.md) · [`xrUICore/Static/UIStatic.h`](../xrUICore/Static/UIStatic.h.md) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — reached through its declarations in [`EliteDetector.h`](EliteDetector.h.md); callers name that, not this file.
**Tier floor** — T2: a world-to-screen projection and a widget tree drawn through a world transform

## Purpose

Two detector models and the readout they share. The readout is the notable thing: it is a
normal widget tree — the same windows and images the menus are built from, laid out from
the same kind of XML — but it is drawn *in the world*, transformed onto a named bone of
the detector's own first-person model, so that the device has a screen the player looks at
by tilting it rather than a flat overlay.

That decision has consequences a rebuild inherits. The widget layer must be able to draw
through an arbitrary world matrix and with lighting, not just in screen space; the
clipping rectangle it uses must be re-derived from that matrix each frame; and the face
culling must be disabled while drawing and restored afterwards, because the screen can be
seen from behind.

The plan view itself is a straightforward top-down projection: artefact positions are
transformed into a frame that follows the camera's heading but not its pitch, scaled so
the sensing radius fills the display, and drawn as marks.

## State

```text
RECORD EliteDetectorUI
  parent            : EliteDetector
  work_area         : widget            # the display's inner rectangle; marks are placed in it
  palette           : map<text, widget> # mark images, keyed by an identifier
  map_attach_offset : transform         # position and rotation of the display relative to its bone
  items_to_draw     : list<(widget, world position)>   # rebuilt every frame
```

Invariant: `items_to_draw` is cleared and refilled every frame; a mark widget from the
palette may appear in it any number of times, so the palette's widgets are drawn at many
positions rather than cloned per contact.

## `CEliteDetector`

**Contract** — the plain model. Fixes its rank ceiling at 3 (it sees every shipped
artefact) and names `elite` as its layout block. Its per-frame work is to rebuild the mark
list from the contact set.

## `UpdateAf` — building the plan

**Contract** — clears the mark list and re-registers one mark per contact, skipping any
artefact currently held by someone. While iterating, an artefact that is capable of being
invisible and is within the forced-visibility radius is made visible: this is the detector's
second function, and the one the player notices most — walking a detector over a hidden
artefact reveals it.

```text
FUNCTION update_marks()
  readout.clear()
  FOR EACH artefact IN contact set
    IF artefact has a parent THEN CONTINUE     # in someone's pocket: not on the map
    readout.register_mark(artefact.position, "af_sign")
    IF artefact can be invisible
       AND distance(detector.position, artefact.position) < vis_radius THEN
      artefact.set_visible(true)
```

**Notes** — the plain model draws every artefact with the same mark; the scientific model
below uses the artefact's section name as the mark identifier instead, so it draws each
kind differently. That single substitution is the whole difference between the two
readouts.

## `render_item_3d_ui` / `render_item_3d_ui_query`

**Contract** — the hooks by which a held item contributes geometry to the first-person
render pass. The query answers whether there is anything to draw — the detector's own
working state. The draw runs the base's contribution, then the readout, then restores the
renderer's face-culling mode, because the readout turned it off.

**Invariants** — the cull-mode restore is not optional: the readout is drawn with culling
disabled so its screen is visible from both sides, and everything drawn afterwards in the
frame would otherwise inherit that.

## `CUIArtefactDetectorElite::construct`

**Contract** — builds the readout's widget tree from the shared detector layout file, using
the owning model's layout tag to select its block. Builds the work area, then the mark
palette: either a set of named palette entries, each an identifier and an image, or — when
the block declares none — a single default mark. Finally reads the display's offset and
rotation relative to its bone from the *detector's own* configuration section, converting
the authored degrees to radians.

**Notes** — the offset living in the item's section rather than in the layout XML is the
seam between the two data sources: the layout file describes what the screen shows, the
item section describes where on the physical model the screen is. A rebuild that merges
them will have to pick one, and the item section is the better home because it is
per-model.

## `CUIArtefactDetectorElite::Draw`

**Contract** — draws the readout in the world. Switches the widget layer to its lit vertex
type, installs the bone-derived world transform, disables culling, draws the widget tree,
then projects each registered mark into the display's plane and draws it there. Restores
the vertex type on the way out.

```text
FUNCTION draw()
  world = locator_matrix()              # bone transform times the authored offset
  saved_point_type = ui.point_type
  ui.point_type = lit

  renderer.set_world_transform(world)
  renderer.set_cull_mode(none)
  draw the widget tree                  # frame, background, needle

  # a frame that yaws with the camera but does not pitch or roll with it
  camera_yaw = heading of the camera direction
  view = inverse(transform with heading camera_yaw at the camera position)

  rebuild the widget layer's clip frustum from the work area's absolute rectangle

  FOR EACH (mark, world_position) IN items_to_draw
    local = view applied to world_position
    scale = work_area.height / detector.detect_radius
    place the mark at (local.x * scale + work_area.width / 2,
                       work_area.height - local.z * scale)
    relative to the work area's absolute origin
    draw the mark

  ui.point_type = saved_point_type
```

**Invariants** — the plan is oriented to the camera's heading only. Pitching the view does
not rotate the plan, which is what makes the display readable while looking down at it.

**Notes**

- The scale divides by the *detector's* detect radius, so a detector configured to sense
  further automatically draws a wider area at the same display size. Radius and zoom are
  one number, which is why they never disagree.
- The vertical placement subtracts the work area's full height, putting the player's own
  position at the bottom edge rather than at the centre: the display is a forward-looking
  sector, not a full circle. Contacts behind the player project to negative coordinates
  and fall outside the work area.
- The per-mark bounds test is present but disabled — every mark is drawn whether or not it
  lands inside the work area, and the frustum installed above is what actually clips. A
  rebuild should use one mechanism.

## `GetUILocatorMatrix`

**Contract** — composes the display's world transform: the held item's own transform, times
the current pose of the model's bone named `cover`, times the authored offset. Re-derived
every frame, so the display tracks the device's animation exactly.

**Notes** — the bone name is a contract with the shipped art. Every detector model that
uses this readout must have a bone called `cover`.

## `RegisterItemToDraw`

**Contract** — appends one mark, looked up in the palette by identifier. An identifier with
no palette entry is logged and skipped rather than faulted, because the scientific
detector keys the palette by artefact section name and the shipped layout does not
necessarily name every section.

## `Clear`

**Contract** — empties the per-frame mark list.

## `CScientificDetector`

**Contract** — the top model. Carries a second detect list over anomaly zones alongside the
artefact list, loaded from the same section under a different key prefix. It senses zones
on the same schedule and at the same radius as artefacts, releases them on the same
transitions, and draws both onto the same readout.

Its per-frame readout refresh replaces the base's rather than extending it, because it
needs to interleave two sources and because it keys marks by section name:

```text
FUNCTION update_work()
  readout.clear()
  FOR EACH artefact IN artefact contacts
    IF artefact has a parent THEN CONTINUE
    readout.register_mark(artefact.position, artefact.section_name)
    reveal it if invisible and within vis_radius     # as the base model does
  FOR EACH zone IN zone contacts
    readout.register_mark(zone.position, zone.section_name)
  readout.update()
```

**Invariants** — the zone contact set must be cleared when the detector leaves the player's
hands and released when it is destroyed, exactly as the artefact set is. Both are done, and
both are the conformance invariant about dangling references.

**Notes** — the zone list is refreshed *unconditionally* while the detector has a carrier,
even when the detector is holstered and not working, whereas the artefact list is refreshed
only while working. Nothing depends on the difference, since the readout is not drawn when
holstered, but it means a holstered scientific detector still costs a proximity query.

## Could not recover

- A helper that rescales a widget tree horizontally by a factor is defined here and never
  called; presumably an aspect-ratio correction that was solved elsewhere.
- Both models fix their artefact rank ceiling at 3 in code rather than reading it from
  configuration, so the ceiling is not tunable and the rank mechanism has no shipped case
  where it actually excludes anything at this tier.
