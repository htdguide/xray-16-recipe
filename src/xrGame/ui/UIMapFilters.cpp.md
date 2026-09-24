# src/xrGame/ui/UIMapFilters.cpp

> The map's marker filter panel: four optional check boxes, a keyboard mode that locks
> navigation into them, and one notification telling the map to rebuild its markers.

**Needs** — [`UIMapFilters.h`](UIMapFilters.h.md) · [`UIHelper.h`](UIHelper.h.md) · [`xrUICore/Buttons/UICheckButton.h`](../../xrUICore/Buttons/UICheckButton.h.md) · [`xrUICore/ui_focus.h`](../../xrUICore/ui_focus.h.md)
**Used by** — [`UIMapFilters.h`](UIMapFilters.h.md)
**Tier floor** — T3.

## Purpose

A small panel with an outsized job: it is the chapter's clearest worked example of the
toolkit's **navigation locker**, and it must work on a layout document that may not mention it
at all. The shipped games' map screens disagree about whether filters exist and about where
they sit, so every element here is optional and the panel can rebuild its own geometry from
its children when the document gives it none.

## State

```text
RECORD FilterPanel EXTENDS Window
  filters   : list<optional<CheckButton>>   # indexed by filter kind; entries may be absent
  activated : bool                          # true while the panel owns the keyboard
```

**Invariants**

- **An absent filter reads as enabled.** A document that omits a check box does not hide that
  kind of marker; it declines to offer the choice. Any other default would make the two games
  whose documents omit filters show an empty map.
- Every check box starts checked, and nothing persists the state across a screen open.
- `activated` mirrors two toolkit facts — this panel holds the keyboard capture, and this
  panel is the navigation locker — and all three are set and cleared together. Losing either
  one without the flag leaves the panel silently deaf.

## `Init`

**Contract** — apply the `filters_wnd` section to this window if the document has one, then
create each of the four check boxes from its own element name, pointing each one's
notifications at the panel, naming it after its element and checking it. Report whether at
least one was created. When the document had no panel section but did define check boxes,
synthesise the panel's rectangle from theirs and rebase them into it.

```text
FUNCTION init(doc)
  had_section <- apply_optional(doc, "filters_wnd", self)
  FOR EACH (kind, element) IN [(treasures,       "filter_treasures"),
                               (quest_npcs,      "filter_quest_npcs"),
                               (secondary_tasks, "filter_secondary_tasks"),
                               (primary_objects, "filter_primary_objects")]
    box <- create_check_optional(doc, element, parent: self)
    IF box IS none THEN CONTINUE
    box.message_target <- self;  box.name <- element;  box.checked <- true

  IF any box was created AND NOT had_section THEN
    self.rect <- the bounding rectangle of every box's own rectangle
    FOR EACH box: box.position <- box.position - self.absolute_position
  RETURN any box was created
```

**Notes** — the geometry synthesis exists because a document may declare the check boxes as
children of the *map screen*, in absolute terms, with no grouping element. Adopting them into
a panel changes what their positions are relative to, so the panel computes its own bounds and
then subtracts them back out. A rebuild whose layout format always groups does not need this.

## `Activate` — the keyboard mode

**Contract** — entering takes the keyboard capture from the panel's message target, installs
the panel as the navigation locker, and puts the focus on the first filter. Leaving releases
both — each only if the panel still holds it — and then tells the message target that the
keyboard capture was lost, so the screen can restore its own bindings.

**Notes** — the locker is *not* a disable: everything outside the panel stays enabled and
merely becomes ineligible for directional navigation. That is what lets the map keep animating
behind an open filter panel. The release path checks ownership before releasing because the
panel can be deactivated by a notification that fires *because* something else already took
the capture; releasing unconditionally would then steal it from the new owner.

## `OnKeyboardAction`

**Contract** — while inactive, the panel answers only the bound *toggle filters* action, which
activates it. While active it first offers the key to its children, then interprets the
bound actions: back or quit deactivates; accept presses the focused filter as though clicked;
and the two generic screen actions are swallowed so the map underneath does not act on them.
Anything else is declined.

**Notes** — the accept path synthesises a *pointer press* on the focused check box rather than
calling its toggle. The check box's press state machine is what emits the click notification,
so going through it is the only way a keyboard accept produces the same downstream effect as a
mouse click. Actions are looked up first in the screen key context and then in the global one,
so a screen-specific binding wins.

## `SendMessage`

**Contract** — a click on any filter re-emits a single "reload the filters" notification to the
panel's message target; the panel does not say which filter changed, because the map rebuilds
all its markers anyway. Losing the keyboard capture, or losing focus while active, deactivates
the panel.

## `IsFilterEnabled` / `SetFilterEnabled`

**Contract** — read and write one filter's check state; an absent filter reads enabled and
ignores writes.
