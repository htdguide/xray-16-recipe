# src/xrGame/ui/UIAchievements.cpp

> An achievement row that polls a script predicate every frame and inserts or removes itself
> from its list accordingly.

**Needs** — [`UIAchievements.h`](UIAchievements.h.md) · [`UIHelper.h`](UIHelper.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`../../xrUICore/ScrollView/UIScrollView.h`](../../xrUICore/ScrollView/UIScrollView.h.md) · [`../../xrUICore/Hint/UIHint.h`](../../xrUICore/Hint/UIHint.h.md) · [Seam: Script virtual machine](../../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — [`UIAchievements.h`](UIAchievements.h.md)
**Tier floor** — T3: a widget polling a script predicate

## Purpose

Achievements are entirely a script concept: the engine does not know what an achievement is,
only that a named Lua function returns true or false. This widget is the whole of the engine's
participation — it holds the function's name and it owns its own membership of the list it is
displayed in.

That inversion is the decision. The list is not filtered by its owner; each candidate row
decides for itself whether to be in it.

## State

```text
RECORD AchievementRow
  list         : the scrolling list this row belongs in when earned
  name, descr, icon : widgets
  hint         : tooltip window, owned outright
  predicate    : text        # the name of a script function returning bool
  repeatable   : bool        # poll even while already in the list
```

Invariant: the row is either in the list or not, and `ParentHasMe` — a linear search of the
list's children — is the only authority on which. There is no cached membership flag, because
the list can be cleared from elsewhere.

## `Update`

**Contract** — Called every frame while the enclosing page is shown.

```text
FUNCTION update()
  # A non-repeatable achievement that is already displayed is settled:
  # never poll again this session.
  IF this row is in the list AND NOT repeatable THEN RETURN

  earned = script(predicate)()          # required; absence is fatal
  IF earned
    IF not in the list THEN join it and show
  ELSE
    IF in the list THEN leave it and hide
```

**Invariants** — The early exit is what keeps the cost bounded: once earned, a normal
achievement costs nothing. A **repeatable** achievement is polled forever, and is the case
where an achievement can be *lost* again — the shipped data uses it for standing-based awards
that come and go.

The predicate's absence is fatal rather than treated as "not earned", because a page listing
an achievement whose script is missing would silently show an incomplete list.

## `SetDescription`

**Contract** — Set the description, wrap it, and **grow the row** so the text fits: the row's
height becomes the description's wrapped height plus 30, but never shrinks below the authored
height.

**Notes** — The 30 is the space the name and icon occupy above the description. It is frozen
against the shared row layout; a rebuild that authors the row differently owes a different
number, and should prefer measuring to a constant.

## `DrawHint`

**Contract** — Draw the tooltip **only when the cursor is inside this row's absolute
rectangle**. Every row's draw is called, so without the test every row in the list would draw
its tooltip at once. This is the game layer supplying the overlap discipline the toolkit's
hover deliberately omits (chapter 15).

## `init_from_xml`

**Contract** — Configure the row and its four parts from one shared element, scoping the
document's local root to that element for the child reads and restoring it afterwards. The row
starts hidden: it becomes visible only by being added to the list.

## `SetName` / `SetHint` / `SetIcon` / `SetFunctor` / `SetRepeatable`

**Contract** — Plain setters. Name and hint go through the localization string table; icon is
a registry name; the functor is stored as text and resolved on every poll rather than bound
once, so a script reloaded mid-session takes effect.

## `Reset`

**Contract** — Leave the list and hide. Called when the UI is reset, which is how the page
starts empty after a level change and lets every row re-decide.
