# src/xrUICore/ui_focus.cpp

> The gamepad and keyboard navigation model — it keeps the set of widgets that can currently take focus, and answers "which widget is next, in this direction" geometrically rather than from an authored tab order.

**Needs** — [`ui_focus.h`](ui_focus.h.md) · [`Windows/UIWindow.h`](Windows/UIWindow.h.md) · [`Cursor/UICursor.h`](Cursor/UICursor.h.md) · [`ui_base.h`](ui_base.h.md) · [Seam: Debug overlay UI](../../SYSTEM-REQUIREMENTS.md#seam-debug-overlay-ui)
**Used by** — [`ui_focus.h`](ui_focus.h.md)
**Tier floor** — T3: set membership and a nearest-neighbour search over a few dozen points.

## Purpose

The shipped screens were authored for a mouse and carry no tab order, no focus chains and no
"next widget" fields. Directional navigation therefore has to be *derived*, and this file
derives it: given where focus is now and which way the player pushed, it picks the nearest
focusable widget lying in that direction.

The second thing it does, and the reason it is a registry rather than a function, is decide
*which* widgets are eligible right now. Eligibility changes constantly — screens are shown
and hidden, a modal takes over — so the registry is re-partitioned every frame rather than
being maintained by whoever changes a flag.

The system does not move a focus highlight. It moves the **cursor**: focusing a widget warps
the pointer onto it, so a gamepad player drives the same mouse-shaped UI a mouse player does.
That is the design's central trick and it is why there is no separate focused-appearance
concept anywhere in the toolkit.

## State

```text
RECORD FocusSystem
  valuable      : list<Window>     # registered AND currently eligible
  non_valuable  : list<Window>     # registered but not eligible right now
  focused       : optional<Window> # where focus is; cleared when it stops being hovered
  locker        : optional<Window> # while set, only its subtree is eligible
```

**Invariants**

- The system **does not own** the windows it holds. A window unregisters itself when it dies;
  failing to do so leaves a dangling entry, which is why unregistration also clears the
  focused pointer if it names the departing window.
- A window is in exactly one of the two lists, never both and never twice. Registration
  always lands in `non_valuable`; the per-frame update moves it.
- `focused` is only meaningful while the cursor is actually over that window. The update
  clears it otherwise, because the cursor may have been moved by the mouse and focus must
  follow the pointer, not fight it.

## `FocusDirection`

**Contract** — nine values: the eight compass directions plus `Same`, which means two
widgets share a centre point. `Same` is a real case (a widget stacked exactly on another)
and is classified rather than being treated as an error.

## `RegisterFocusable` / `UnregisterFocusable` / `IsRegistered` / `IsValuable` / `IsNonValuable`

**Contract** — registration is idempotent and always enters the non-eligible list;
the next update promotes it if it qualifies. Unregistration removes from whichever list holds
it and drops the focus if it was focused. The three predicates are membership tests.

**Notes** — most widgets register themselves on construction, and several then *unregister a
child* immediately afterwards. A scroll bar's arrow buttons are the clearest case: they are
buttons, so they self-register, but a gamepad player should not be able to focus them — the
scroll bar's owner handles scrolling as a whole. Unregistering is how a container overrides a
child's self-registration, and a rebuild that registers from the outside instead avoids the
dance entirely.

## `Update`

**Contract** — re-partitions the two lists against the currently active tree root, then
drops the focus if the cursor has wandered off the focused window. Called once per frame by
whoever owns the active screen, passing that screen as the root.

```text
FUNCTION update(sys, root)
  demoted = empty
  FOR EACH w IN sys.valuable
    IF NOT is_focus_valuable(w, root, sys.locker)
      remove w from valuable; append w to demoted
      IF w IS sys.focused THEN sys.focused = none

  FOR EACH w IN sys.non_valuable
    IF is_focus_valuable(w, root, sys.locker)
      remove w from non_valuable; append w to valuable

  append every window in `demoted` to non_valuable

  IF sys.focused EXISTS AND cursor is not over sys.focused
    sys.focused = none
```

`is_focus_valuable` is the window's own ancestor walk (see
[`UIWindow.cpp`](Windows/UIWindow.cpp.md)): shown and enabled all the way up, inside the
locker's subtree if there is one, and rooted at the expected screen.

**Invariants** — demotions are collected and appended *after* the promotion pass, so a window
demoted this frame is not immediately re-examined for promotion in the same frame. The
scratch list is stack-allocated at the exact size needed, which is the original's way of
saying: this runs every frame and must not allocate.

**Notes** — the promotion and demotion loops erase from the list they are iterating and then
dereference the returned iterator. That pattern is fragile in the original language and the
demotion loop in particular compares the *next* element against the focused window rather
than the one just removed. Read the intent, not the code: the window that was demoted is the
one that must lose focus.

## `SetFocused`

**Contract** — sets the focused window *and warps the cursor onto its centre*. There is no
way to focus something without moving the pointer, by design.

## `FindClosestFocusable`

**Contract** — given a point and a requested direction, returns two candidates: the nearest
eligible widget lying exactly in that direction, and the nearest lying in either of the two
directions adjacent to it. Either may be absent. The caller decides which to take — normally
the primary, falling back to the secondary when nothing lies straight ahead.

```text
FUNCTION find_closest(sys, from, direction) -> (primary, secondary)
  (left_of, straight, right_of) = allowed_directions(direction)
  primary = none; secondary = none
  best = infinity; best2 = infinity

  FOR EACH w IN sys.valuable
    to  = absolute centre of w
    d   = squared distance from `from` to `to`
    dir = classify(from, to)                 # one of the nine directions
    IF d < best  AND dir == straight
      best = d;  primary = w
    IF d < best2 AND (dir == left_of OR dir == right_of)
      best2 = d; secondary = w
  RETURN (primary, secondary)
```

`allowed_directions` widens a requested direction into the three that count as "that way":
`Up` admits up, upper-left and upper-right; `Left` admits left, upper-left and lower-left;
a diagonal request admits the diagonal plus the two axes that bound it. The *straight* member
of the triple is always the requested direction itself.

`classify` compares two centre points and returns the compass direction, treating exactly
equal coordinates as axis-aligned rather than diagonal, and coincident points as `Same`.

**Invariants**

- Distances are compared squared; no square root is taken, because only the ordering matters.
- Requesting `Same` as a direction is a caller error; the code asserts and then behaves as if
  `Down` had been requested, on the reasoning that down is the natural "enter into something"
  direction. That fallback is arbitrary but harmless.
- Widgets are compared by their **absolute centre points**, not their rectangles. A wide
  widget therefore navigates as if it were a point at its middle, which is why a long row of
  narrow buttons navigates well and a screen mixing very large and very small widgets does
  not always pick what the player expected.

**Notes** — the search is linear over every eligible widget. With a few dozen candidates that
is free; a rebuild should not be tempted into a spatial index it would then have to keep in
step with a tree that changes every frame.

## `LockToWindow` / `Unlock` / `GetLocker`

**Contract** — while a locker is set, only windows inside its subtree are eligible. This is
how a modal popup captures navigation without disabling anything else: nothing about the rest
of the screen changes, it simply stops being reachable. Used by the context menu — see
[`UIPropertiesBox.cpp`](PropertiesBox/UIPropertiesBox.cpp.md).

## Debug tree, debug inspector, direction overlay

**Contract** — the focus system presents itself to the debug overlay as two lists, eligible
and not, each expandable into the same window tree the main debug view shows. When one
eligible window is hovered in that tree, the overlay draws a line from it to every other
eligible window, labelled with the compass direction the classifier assigns — which is the
only practical way to see why navigation chose what it chose. Compiled out of the shipping
build.
