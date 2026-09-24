# src/xrUICore/Windows/UIWindow.cpp

> The node of the retained widget tree — it owns the parent/child relation, the relative-coordinate and alignment rules, the draw and hit-test order, mouse and keyboard capture, and the message path a control uses to tell its owner something happened.

**Needs** — [`UIWindow.h`](UIWindow.h.md) · [`UIMessages.h`](../UIMessages.h.md) · [`uiabstract.h`](../uiabstract.h.md) · [`ui_debug.h`](../ui_debug.h.md) · [`ui_base.h`](../ui_base.h.md) · [`Cursor/UICursor.h`](../Cursor/UICursor.h.md) · [Seam: Debug overlay UI](../../../SYSTEM-REQUIREMENTS.md#seam-debug-overlay-ui)
**Used by** — [`UIWindow.h`](UIWindow.h.md)
**Tier floor** — T3: a tree of records with pointer identity and event dispatch; nothing here touches a device, a byte layout or a budget. The only reason it is not written in the most abstract tier available is that it is walked once per frame for every visible widget.

## Purpose

Every player-facing control in the engine is this type or derives from it. The file settles
five questions that every other widget then inherits rather than re-deciding: where a
window is in screen space, who draws before whom, which window a click belongs to, who
owns a child's memory, and how a child tells its owner that something happened.

It is a *retained* toolkit, in deliberate contrast to the immediate-mode toolkit used for
the debug overlay: the tree persists across frames, so a control can hold state (a pushed
button, a scroll position, a text cursor) and the frame loop only walks it.

## State

```text
RECORD Window
  name             : text            # identity for lookup and for the debug tree
  position         : vec2            # relative to the parent's top-left, in canvas units
  size             : vec2
  alignment        : Alignment       # how position is interpreted; see GetWndRect
  visible          : bool
  enabled          : bool            # "may receive input"; drawn state is `visible`
  cursor_over      : bool            # recomputed each Update from the cursor position
  custom_draw      : bool            # parent must not draw me; someone else will
  auto_delete      : bool            # my memory belongs to whoever detaches me
  parent           : optional<Window>
  children         : list<Window>    # order is draw order: later = on top
  mouse_capturer   : optional<Window>   # a *child*, not an arbitrary window
  key_capturer     : optional<Window>   # likewise
  message_target   : optional<Window>   # where my notifications go; default is parent
  last_cursor_pos  : vec2            # in my own coordinates, from the last mouse event
  last_click_time  : int             # for synthesising the double click
  focus_time       : int             # when the cursor entered me; 0 when it is outside
```

**Invariants**

- A window appears in at most one parent's child list, and `parent` and that list agree.
  Attaching asserts the child is not already mine; setting a parent asserts the old parent
  has already forgotten me.
- `mouse_capturer` and `key_capturer` name a *direct child*. Capture is therefore a chain:
  a leaf that captures causes every ancestor up to the root to capture the next link, so
  the root can route an event straight down to the leaf in one hop per level.
- A window that is `auto_delete` is destroyed by its parent's detach, never independently.
  The reverse case — a window destroyed while still parented and marked auto-delete — is a
  double ownership claim and is asserted against.
- Canvas coordinates are always the virtual 1024×768 space (see
  [`ui_defs.h`](../ui_defs.h.md)); nothing in this file knows the real resolution.

## `Window` — geometry

**Contract** — position and size are plain fields with trivial accessors; the interesting
one is the *rectangle*, which is not simply `(position, position + size)` because of
alignment.

`GetWndRect` derives the window's rectangle in its parent's coordinates from position,
size and alignment mode:

```text
FUNCTION rect_of(w) -> Rect
  IF w.alignment IS None OR Left
    RETURN rect(w.position, w.position + w.size)
  IF w.alignment IS Center
    RETURN rect(w.position - w.size/2, w.position + w.size/2)   # position is the centre
  IF w.alignment IS Right
    # pinned to the right edge of the canvas; only y comes from position
    RETURN rect(CANVAS_WIDTH - w.size.x, w.position.y, CANVAS_WIDTH, w.position.y + w.size.y)
  IF w.alignment IS Top
    RETURN rect(w.position.x, 0, w.position.x + w.size.x, w.size.y)
  IF w.alignment IS Bottom
    RETURN rect(w.position.x, CANVAS_HEIGHT - w.size.y,
                w.position.x + w.size.x, CANVAS_HEIGHT)
```

`GetAbsoluteRect` walks to the root adding each ancestor's rectangle origin, then re-imposes
this window's own size — so a window is positioned relative to its parent but **not clipped
or resized** by it. Clipping is a separate, explicit decision made by the few containers
that want it (see [`UIScrollView.cpp`](../ScrollView/UIScrollView.cpp.md)).

**Notes** — the alignment modes for Right, Top and Bottom silently discard one component of
`position`. That is the point: a window declared with one of those alignments in XML sticks
to the screen edge at any aspect ratio, and the layout author writes only the free
coordinate. The Right case measures against the canvas width, not the parent, so it pins to
the *screen* edge even for a nested window — which is a quirk a rebuild should preserve,
because the shipped layouts rely on it.

## `AttachChild` / `DetachChild` / `DetachAll`

**Contract** — attach appends to the child list (therefore on top of every existing sibling)
and takes the parent link. Detach removes the link, releases mouse capture if the departing
child held it, and — if the child is marked auto-delete — destroys it. `DetachAll` repeatedly
detaches the last child, so a tree is torn down leaves-last-attached-first.

**Invariants** — detaching a window that is not a child is a programming error, not a
tolerated no-op. Keyboard capture is *not* cleared on detach, which is an asymmetry with
mouse capture and a latent stale pointer; a rebuild should clear both.

## `Draw`

**Contract** — draws children in list order, skipping those that are not shown and those
marked custom-draw. Does not draw itself: the base window is invisible, and every visible
widget overrides this and calls back into it to get its children painted. List order is
painter's order, so the last-attached child is on top.

## `Update`

**Contract** — recomputes cursor-over state from the cursor's absolute position against
this window's absolute rectangle, raises the enter/leave transition, then recurses into
shown children. This is the *hover* notion, distinct from the navigation focus owned by
[`ui_focus.cpp`](../ui_focus.cpp.md).

```text
FUNCTION update(w)
  over = cursor_visible AND absolute_rect(w) CONTAINS cursor_position
  IF over != w.cursor_over
    IF over THEN on_focus_receive(w) ELSE on_focus_lost(w)
  FOR EACH c IN w.children
    IF c.visible THEN update(c)
```

`on_focus_receive` stamps the current time and notifies the message target; `on_focus_lost`
clears the stamp. The stamp is what delayed hints are measured from — see
[`UIStatic.cpp`](../Static/UIStatic.cpp.md), which waits 700 ms of hover before showing one.

**Notes** — the hover test runs against the *whole* absolute rectangle with no regard to
overlapping siblings, so two overlapping windows can both consider themselves hovered. The
mouse *routing* below does resolve overlap; hover does not. Layouts are authored not to
overlap, which is why this is survivable.

## `OnMouseAction`

**Contract** — the single entry point for every pointer event. Takes a position in this
window's own coordinates (except at the root, where it takes canvas coordinates and
converts), plus the event kind. Returns whether the event was consumed. Called top-down from
the engine's input pump each time the cursor moves, a button changes, or the wheel turns.

```text
FUNCTION on_mouse_action(w, x, y, action) -> bool
  w.last_cursor_pos = (x, y)

  # Synthesise the double click here rather than in the input layer, so that
  # every widget gets one for free and none of them has to keep a timer.
  IF action IS LEFT_DOWN
    IF this_frame_has_not_already_produced_a_double_click
       AND now - w.last_click_time < DOUBLE_CLICK_WINDOW      # 250 ms
      action = LEFT_DOUBLE_CLICK
      mark this frame as having produced one
    w.last_click_time = now

  IF w IS root
    IF NOT rect_of(w) CONTAINS (x, y) THEN RETURN false
    subtract rect_of(w).top_left FROM (x, y)                  # to local coordinates

  IF w.mouse_capturer EXISTS
    # capture wins unconditionally, even when the cursor has left the child
    forward to w.mouse_capturer with coordinates rebased to it
    RETURN true

  dispatch action to w's own handler (move / wheel / button-down / double-click)
  IF the handler consumed it THEN RETURN true

  # Children in REVERSE order: the topmost-drawn gets first refusal.
  FOR EACH c IN reverse(w.children)
    IF NOT c.enabled THEN CONTINUE
    IF rect_of(c) CONTAINS (x, y) OR c.cursor_over
      IF on_mouse_action(c, rebased x, rebased y, action) THEN RETURN true

  RETURN false
```

**Invariants** — hit-test order is the exact reverse of draw order. That is the whole
z-order model: there is no explicit depth, no sorting, only list position.

**Notes**

- The second half of the child test — `OR c.cursor_over` — delivers events to a child the
  cursor has just *left*. Without it a child could never learn that the pointer went away
  while a button was held, and a dragged scroll box would freeze at the window edge. The
  original source calls this condition "very strange code"; it is load-bearing.
- The double-click guard is per *frame*, not per window: one physical press must not be
  reinterpreted as a double click by several windows in the same frame. A rebuild that
  synthesises the double click in the input layer instead gets this for free.
- The 250 ms threshold is the conventional desktop value and is not derived from anything
  in the game data.

## `SetCapture` / `SetKeyboardCapture`

**Contract** — a child asks its parent to route all further pointer (or key) events to it
regardless of position. The request walks up: each level captures the link below it, so the
root ends up holding a direct path down to the capturer. Releasing walks up the same way.
When a capture displaces a previous capturer, the displaced window is told so by message, so
it can abandon whatever gesture it was in the middle of.

**Notes** — the controller path reuses the keyboard capturer rather than having its own; the
source flags this as a known shortcut. A rebuild with gamepad support should decide whether
the two are really the same channel.

## `OnKeyboardAction` / `OnTextInput` / `OnControllerAction`

**Contract** — all three share one shape: offer the event to the capturer first, then to
children in reverse order, stopping at the first that consumes it. None of them consults
position, and none of them has a root-level coordinate conversion, so the three are pure
tree walks.

Text input is a *separate channel* from key events: the windowing seam delivers composed
characters independently of scancodes (see
[Seam: Windowing and input](../../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)), and
an edit box consumes characters from the one and navigation from the other. Conflating them
breaks non-Latin input.

## `SendMessage`

**Contract** — the notification channel, orthogonal to input. A window sends a
`(sender, message-id, payload)` triple to its *message target*, which defaults to its parent
but can be redirected. The default implementation broadcasts the triple down to every enabled
child, so a container can rebroadcast without knowing who cares.

**Notes** — the message vocabulary is the frozen enumeration in
[`UIMessages.h`](../UIMessages.h.md). The default-to-parent rule is what makes a control
reusable: a button does not know who owns it, it just notifies upward, and a container that
wants a grandchild's notifications sets itself as that grandchild's target.

## `GetCurrentMouseHandler` / `GetChildMouseHandler`

**Contract** — answers "which window would the next pointer event land in", by running the
same reverse-order descent as `OnMouseAction` against the last recorded cursor position, but
without dispatching anything. Used by tooling and by controls that need to know whether they
are about to lose the pointer.

## `IsFocusValuable`

**Contract** — decides whether this window may currently receive *navigation* focus (the
gamepad/keyboard focus managed by [`ui_focus.cpp`](../ui_focus.cpp.md)), given the tree root
that is currently active and an optional locking window.

```text
FUNCTION is_focus_valuable(w, expected_root, locker) -> bool
  node = w
  LOOP
    valuable = node.visible AND node.enabled
    IF node IS locker THEN remember that w is inside the locker's subtree
    IF NOT valuable OR node HAS NO parent THEN BREAK
    node = node.parent
  # node is now either the first hidden/disabled ancestor, or the root

  IF locker EXISTS AND w is not inside the locker's subtree THEN RETURN false
  IF expected_root EXISTS AND node != expected_root THEN RETURN false
  RETURN valuable
```

**Invariants** — a window is focusable only if *every* ancestor up to the root is both shown
and enabled, and only if it belongs to the screen that is actually on top. The `locker` is
how a modal popup takes over navigation without anyone else being disabled: it does not
change any flag, it simply makes every window outside its subtree non-valuable.

## `FindChild` / `GetTop` / `GetWindowBeforeParent`

**Contract** — name lookup walks the subtree depth-first, matching this window's own name
first, and returns the first match (names are not required to be unique; the first one wins).
`GetTop` returns the root of the tree. `GetWindowBeforeParent(ancestor)` returns the
*direct child of `ancestor`* on the path from this window up — the "which row of the list am
I in" question, which is how a scroll view maps a deeply nested focused widget back to the
row it must scroll into view.

## `Reset` / `ResetAll`

**Contract** — drops both capture pointers, returning the window to a state where no gesture
is in progress. `ResetAll` does this to every direct child. Called when a screen is shown or
hidden so a half-finished drag does not survive into the next appearance.

## `Show` / `Enable` / `ShowChildren`

**Contract** — `Show` sets visibility *and* enabled together, because a hidden-but-enabled
window would still answer to input the player cannot see. `Enable` alone separates the two
for controls that want to be visible but inert (a greyed-out button). `ShowChildren`
applies `Show` one level down only.

## `fit_in_rect`

**Contract** — places a window (in practice a hint or tooltip) near the cursor so that it
lies entirely inside a given visible rectangle. Returns false and places nothing if the
cursor itself is outside that rectangle. Free function, not a method, because it is a layout
policy rather than a property of any window.

```text
FUNCTION fit_in_rect(w, visible, border, widescreen_shift) -> bool
  cursor = cursor_position
  IF the display is wider than the canvas aspect
    cursor.x = cursor.x - widescreen_shift    # caller-supplied correction
  IF NOT visible CONTAINS cursor THEN RETURN false

  candidate = rectangle of w's size, inset by `border`, placed at the cursor
  # Try four placements in order, each time only if the previous did not fit:
  #   above the cursor, above-left, below-left, below-right (clearing the cursor glyph)
  # then two last-resort vertical/horizontal pulls back inside `visible`.
  place w at the first candidate fully inside `visible`
  RETURN true
```

**Notes** — the cursor glyph height (43 canvas units) is hard-coded as the amount the
below-right placement must clear so the tooltip does not sit under the pointer. It matches
the shipped cursor art and is not read from the data; a rebuild should take it from the
cursor's actual size.

## Debug tree and debug inspector

**Contract** — every window can describe itself to the debug overlay: a tree node carrying
its name and concrete type, and a property sheet exposing position, size, the flags, the
capture pointers and the message target. Editing the sheet mutates the live window.

The tree walk also draws each window's absolute rectangle as an overlay, colour-coded by
role — hovered, examined in the tree, navigation-focused, focus-valuable, focus-non-valuable
— or by a hash of the window's identity when the "random colours" mode is on. The hash is
seeded from the window's identity precisely so the colour is stable frame to frame rather
than flickering.

**Notes** — this whole surface is compiled out of the shipping build, and a rebuild may
drop it. It is worth keeping, because the alternative way to answer "why did this click go
to the wrong widget" in a tree with no z-order is to read the layout XML by hand.
