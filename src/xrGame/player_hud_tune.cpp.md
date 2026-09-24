# src/xrGame/player_hud_tune.cpp

> Positioning a weapon in the player's hands by eye: live sliders over the eleven authored measurements, the debug markers that show where the muzzle actually is, and the configuration text to paste back.

**Needs** — [`player_hud_tune.h`](player_hud_tune.h.md) · [`player_hud.h`](player_hud.h.md) · [`HudItem.h`](HudItem.h.md) · [`HUDManager.h`](HUDManager.h.md) · [`Level.h`](Level.h.md) · [`debug_renderer.h`](debug_renderer.h.md) · [`xrEngine/xr_input.h`](../xrEngine/xr_input.h.md) · [`xrEngine/CameraManager.h`](../xrEngine/CameraManager.h.md) · [`xrUICore/ui_base.h`](../xrUICore/ui_base.h.md) · [Seam: Debug overlay UI](../../SYSTEM-REQUIREMENTS.md#seam-debug-overlay-ui)
**Used by** — reached through its declarations in [`player_hud_tune.h`](player_hud_tune.h.md); callers name that, not this file.
**Tier floor** — T3: an editor panel over live values

## Purpose

Placing a weapon model in first-person hands is not a calculation; it is a judgement made by
looking at it. The eleven numbers that do it have no meaning outside the view they produce,
and each is authored to seven decimal places.

This tool is the way those numbers were arrived at. It is compiled out of release builds, and
it is the only part of the game where a developer's workflow is visible in the source — which
makes it worth reading for what it says about the data: **every one of those measurements was
tuned by hand, per weapon, per aspect ratio, and none of them is derivable.**

## State

Declared in [`player_hud_tune.h`](player_hud_tune.h.md): the item being edited, two copies of
its measurements — the loaded one and the edited one — the slider step sizes, and five flags
choosing which debug markers to draw.

**Invariants**
- Switching which hand is being edited reloads the measurements from configuration, so edits
  do not leak between items.
- The edited copy is pushed into the live item **every frame**, not on a commit. There is no
  apply button and no undo except reloading from configuration.
- The tool's active state is polled by the rig (see
  [`player_hud.cpp`](player_hud.cpp.md)), which rebuilds the item's attachment transform from
  the edited values each frame instead of from the loaded ones. That poll is the whole of
  how the edits become visible.

## `on_tool_frame`

**Contract** — draws the panel and applies its edits. Does nothing when the panel is closed
or no rig exists. Compiled out entirely in release builds.

```text
FUNCTION on_tool_frame()
  IF the panel is closed OR no player rig exists THEN RETURN
  item = rig.attached_item(current_hand)
  IF item differs from the one being edited
    current_item = item ; ResetToDefaultValues()

  draw the pause toggle, the hand selector, the field-of-view slider
  draw the two step-size sliders
  draw one three-component slider per tunable measurement, editing the edited copy
  UpdateValues()                       # push into the live item, every frame

  IF an item is being edited
    offer the clipboard export
    draw the requested debug markers
  offer the reset
  draw the bone-visibility and animation panels
```

**Notes** — the **pause** toggle sets the device's time factor to very nearly zero rather
than to zero. A true zero would divide by itself in the frame-delta smoothing; a
near-zero freezes the world while keeping every clock advancing, which is what lets the tool
keep drawing, the sliders keep responding, and an animation still be stepped. That trick is
the reason the tool can be used at all — positioning a weapon requires the world to hold
still and the interface not to.

The two step sizes are themselves sliders, with ranges spanning four orders of magnitude for
position and five for rotation. That is the tool admitting that placement is a
coarse-then-fine process: metres first, then ten-micrometre nudges.

The field-of-view slider edits the heads-up display's own field of view, which is separate
from the world's. Changing it changes how much of the weapon is in frame and therefore where
it must sit, so it belongs beside the placement sliders rather than in a graphics menu.

## The debug markers

**Contract** — draws the weapon's computed muzzle, second muzzle, ejection port and fire
directions as world-space markers, each converted back into the heads-up display's own space
before drawing. Development builds only.

**Notes** — the conversion back into display space is the load-bearing part. The muzzle is
computed in world space (see `setup_firedeps` in
[`player_hud.cpp`](player_hud.cpp.md)) but the weapon is drawn in the display's own
projection with a different field of view, so a world-space marker would not land on the
weapon. Converting makes the marker sit exactly where the artist sees the muzzle, which is
the only way to tell whether the authored offset is right.

The fire direction is drawn as a line from the muzzle to the current aim ray's range, so its
far end lands where a shot would. Comparing that end against the crosshair is how an
off-axis weapon is caught.

## The clipboard export

**Contract** — writes the edited measurements as configuration text, in the shipped format,
with the aspect-ratio suffix applied to the six keys that have one. Straight to the
clipboard.

**Notes** — this is what makes the tool a workflow rather than a toy: the artist positions by
eye and pastes the result into the weapon's configuration file. There is no write-back path,
deliberately — the tool does not know which file a section came from, and the virtual
filesystem's mounted archives are not writable.

The export writes the key name `fire_point` **twice**, once for each fire point; the second
should be `fire_point2`. A paste therefore silently overwrites the first muzzle with the
second. It is a bug in a developer tool with no effect on the game, and a rebuild should
simply fix it.

The item placement keys and the three points are exported without an aspect-ratio suffix,
matching what the loader reads — see
[`player_hud.cpp`](player_hud.cpp.md).

## `ResetToDefaultValues` / `UpdateValues`

**Contract** — `ResetToDefaultValues` re-reads the item's measurements from configuration
and copies them into both the loaded and the edited copy; with no item selected it zeroes
every field instead, so the sliders start somewhere meaningful. `UpdateValues` copies the
edited measurements onto the live item.

**Notes** — pushing every frame rather than on demand means a stale edited copy would
overwrite a legitimate reload. That is why switching hands resets first: the copy must be
refreshed before the next push.

## The bone and animation panel

**Contract** — lists the item model's bones with a visibility toggle each, and every motion
alias as a button that plays it. The root bone is excluded from the list.

**Notes** — bone visibility toggling is how a weapon's optional parts — a scope, a silencer,
a magazine that hides during a reload — are checked against the animation. Excluding the root
prevents hiding the whole model, which would be unrecoverable from inside the tool.

The animation buttons play through the item's *no-callback* motion path, so an animation can
be previewed without firing the game events it normally would — no round is chambered, no
casing ejects, no ammunition is consumed. That distinction is what makes previewing safe
mid-game.

Each alias's tooltip shows the base clip name and the item-model clip name, which is the only
place in the running game where that indirection is visible.
