# src/xrGame/ui/UIWarState.cpp

> One faction-war status slot in the PDA: an icon that appears when a script gives it one, with a
> delayed tooltip, and nothing at all when it does not.

**Needs** — [`UIWarState.h`](UIWarState.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`UIHelper.h`](UIHelper.h.md) · [`xrUICore/Static/UIStatic.h`](../../xrUICore/Static/UIStatic.h.md) · [`xrUICore/Hint/UIHint.h`](../../xrUICore/Hint/UIHint.h.md)
**Used by** — [`UIWarState.h`](UIWarState.h.md)
**Tier floor** — T3: a picture with a visibility rule

## Purpose

The PDA's faction pages show a row of small status icons whose meaning is entirely script-defined:
the engine provides the slots, a script fills them. A slot with nothing in it is not an empty box —
it is invisible, and contributes no tooltip.

The whole file is the "fill or hide" rule plus a tooltip delay read from the layout.

## State

```text
RECORD WarStateSlot extends HintWindow
  icon : Widget      # the picture; the tooltip lives in the base
```

Invariants:

- A slot with no icon name is invisible and carries an empty tooltip. Visibility and tooltip content
  are always set together; there is no state where one is stale relative to the other.

## `InitXML`

**Contract** — Attaches itself to the given parent, marks itself for automatic deletion, configures
itself from the named element, builds the icon from that element's `img` child, and reads the
tooltip's dwell delay from the element's `delay` attribute (defaulting to none).

**Notes** — This is the unusual shape in the chapter: the widget attaches *itself* to the parent
rather than being attached by the caller, which is why the call takes a parent at all. It lets a
caller build a whole row of slots in a loop with one call each.

## `UpdateInfo`

**Contract** — Shows the slot with the named icon and the given tooltip identifier, and reports
success. An empty or missing icon name is refused and reports failure, leaving the slot as it was —
so a caller filling a row can use the return value to stop at the first empty slot.

```text
FUNCTION UpdateInfo(icon_name, hint_id) -> bool
  IF icon_name IS empty THEN RETURN false
  visible <- true
  icon.bind_texture(icon_name)
  hint_text <- localized(hint_id) IF hint_id is non-empty ELSE ""
  RETURN true
```

## `ClearInfo`

**Contract** — Hides the slot and clears its tooltip.

## `Draw`

**Contract** — Draws only when shown. The base hint window would otherwise paint its tooltip for a
hidden slot, because a tooltip is not part of the widget subtree and is not hidden with it.

**Notes** — The file carries a commented-out earlier design in which a slot kept a default texture
and dimmed instead of disappearing, tracked by its own "installed" flag. The shipped behaviour is
the simpler one: hide entirely. The default-texture path is dead and a rebuild should not revive it.
