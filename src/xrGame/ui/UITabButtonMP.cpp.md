# src/xrGame/ui/UITabButtonMP.cpp

> A tab button that always reports itself enabled, nudges its text when hovered, and paints a second
> label whose colour tracks the button's state.

**Needs** — [`UITabButtonMP.h`](UITabButtonMP.h.md) · [`xrUICore/TabControl/UITabButton.h`](../../xrUICore/TabControl/UITabButton.h.md) · [`xrUICore/Static/UIStatic.h`](../../xrUICore/Static/UIStatic.h.md)
**Used by** — [`UITabButtonMP.h`](UITabButtonMP.h.md)
**Tier floor** — T3: state-to-colour and state-to-offset over a button

## Purpose

The multiplayer menus use tab strips where a tab carries two pieces of text — a caption and a
secondary hint line underneath — and where a tab's *visual* disabled state must not make it
unclickable. This button provides both.

The disabled-state handling is the load-bearing oddity. The base button uses one flag for two
distinct questions — "may this be interacted with" and "should this look greyed out" — and this
screen needs them separated. Rather than split the flag, this button reports itself permanently
enabled to callers while still *using* the flag internally to pick colours. The result is a tab that
can look inactive and still respond.

## State

```text
RECORD MultiplayerTab extends TabButton
  text_offset_normal      : Vector2   # where the caption sits at rest
  text_offset_hovered     : Vector2   # where it sits under the cursor
  hint                    : optional<Widget>   # second label; drawn by this button
  vertical                : bool      # set from the layout; read by nothing in this file
```

Invariants:

- `hint` is attached as a child but marked **custom-draw**, which excludes it from the tree's own
  draw pass. It is painted explicitly after the button, so it sits on top of the button face.
- The enabled flag is temporarily forced true around the base update and then restored; nothing
  outside this file may observe it mid-update.

## `IsEnabled`

**Contract** — Returns true unconditionally. This is the decision the whole file exists for: the
tab is always clickable, whatever its visual state.

## `SendMessage`

**Contract** — Any notification whose sender is this button **re-enables** it, then falls through to
the base. A tab that was greyed out becomes visually enabled the moment the player interacts with
it, which is how the multiplayer menus show "this tab now has content".

## `Update`

**Contract** — Runs the base update with the enabled flag forced true while the cursor is over the
button, so the base paints a hovered tab in its enabled colours even when it is nominally disabled;
restores the flag afterwards. Then applies the text offset, and recolours the hint label from the
button's state.

```text
FUNCTION Update()
  saved <- enabled
  IF cursor is over this button THEN enabled <- true
  base.Update()
  enabled <- saved

  caption.offset <- text_offset_hovered IF cursor is over this button ELSE text_offset_normal

  IF hint EXISTS THEN
    state <- disabled    IF NOT enabled
             pushed      IF the button is held down
             highlighted IF the cursor is over it
             enabled     OTHERWISE
    hint.colour <- the button's colour for that state,
                   falling back to the enabled colour where that state has none
```

**Notes** — The fallback to the enabled colour for a state with no configured colour is what lets a
layout give a tab only one colour and still get a sensible hint label in every state. The state
precedence — disabled beats pushed beats hovered — is the order tested, and matters because a
disabled button can still be hovered here.

The text offset shift is a two-position nudge, not an animation: the caption jumps between two
authored positions. A rebuild that interpolates would look different from the original.

## `CreateHint`

**Contract** — Builds the secondary label, attaches it, and marks it custom-draw so the tree skips
it. Called by the layout reader only when the layout actually declares a hint element, so a tab
without one has no label at all rather than an empty one.

## `Draw`

**Contract** — Draws the button face, then the hint label on top. The explicit second call is the
other half of the custom-draw arrangement.

## `SetOrientation`

**Contract** — Records whether the tab strip runs vertically. Nothing in this file reads it; it is
set from the layout and the tab strip above consults it. It is here because the flag is per tab in
the layout vocabulary.
