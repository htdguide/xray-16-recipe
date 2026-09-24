# src/xrGame/UIDialogHolder.cpp

> The screen stack and the input router: it decides which window is modal, which windows draw, what happens to a key the top window does not want, and when the pointer is visible.

**Needs** — [`UIDialogHolder.h`](UIDialogHolder.h.md) · [`ui/UIDialogWnd.h`](ui/UIDialogWnd.h.md) · [`UIGameCustom.h`](UIGameCustom.h.md) · [`Level.h`](Level.h.md) · [`Actor.h`](Actor.h.md) · [`xrUICore/Cursor/UICursor.h`](../xrUICore/Cursor/UICursor.h.md) · [`xrEngine/CustomHUD.h`](../xrEngine/CustomHUD.h.md) · [`xrEngine/xr_level_controller.h`](../xrEngine/xr_level_controller.h.md) · [Seam: Windowing and input](../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input) · [Seam: Debug overlay UI](../../SYSTEM-REQUIREMENTS.md#seam-debug-overlay-ui)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: list management with deferred mutation, and an input dispatch chain

## Purpose

Anything that can show screens is one of these: the main menu, the in-game interface, the
multiplayer lobby. What they all need, and what this supplies, is the arbitration between
*windows* and *the game*.

Two collections, and the difference between them is the central idea:

- **the draw list** — every window currently on screen, modal or not. The health bar, the
  compass and the inventory screen are all in it.
- **the input stack** — only the windows that take input, most recent on top. Exactly one
  window, the top of this stack, receives events at a time.

A window can be in the draw list without being in the input stack (a passive readout), and
the stack is what makes a screen modal. Neither collection owns its windows.

Everything else in the file follows from three rules:

1. **Input falls through.** The top window sees an event first. If it does not consume it,
   and it does not declare that it blocks player movement, the event is re-offered to the
   entity the player is controlling as a game action. That is how you can walk while the
   map is open and cannot while the inventory is.
2. **Opening a screen may hide the heads-up display, and closing it must put back exactly
   what was there.** The previous state is saved *per stack entry*, so nested screens
   unwind correctly.
3. **Both collections may be mutated while they are being walked**, because a window's
   update can open or close another window. Mutation is therefore deferred.

## State

```text
RECORD DrawEntry
  window  : Window
  enabled : bool        # false means "scheduled for removal at end of frame"

RECORD StackEntry
  window          : DialogWindow
  saved_crosshair : bool      # the state this entry hid, to be restored when it closes
  saved_indicators: bool

RECORD DialogHolder
  input_stack   : list<StackEntry>    # top is last
  draw_list     : list<DrawEntry>
  pending_draws : list<DrawEntry>     # additions made while the draw list was being walked
  in_update     : bool
  is_foremost   : bool                # only the foremost holder owns the pointer and focus
  cursor_shown_at : int (ms)          # for the pointer's auto-hide timer
```

**Invariants**

- A window appears at most once in the draw list and at most once in the pending list; both
  are checked before an insertion.
- Removal never erases during a walk. It marks the entry disabled and hides the window; the
  end of the frame sorts disabled entries to the end of the list and pops them.
- The saved heads-up state lives on the stack entry that *caused* the hiding, not in a
  single global slot, which is what makes nesting work.

## `StartMenu` / `StartDialog` / `StartStopMenu`

**Contract** — opens a screen. Requires that it is not already shown. Adds it to the draw
list, pushes it onto the input stack, optionally saves and hides the heads-up display,
shows the pointer if the screen wants one, and disentangles the player from whatever he was
doing.

```text
FUNCTION StartMenu(dialog, hide_indicators)
  REQUIRE dialog is not shown
  add_to_draw_list(dialog)
  push_input_receiver(dialog)

  IF this holder uses indicators AND the stack is not empty THEN
    top_entry.saved_crosshair  <- the crosshair's current state
    top_entry.saved_indicators <- the heads-up display's current state
    IF hide_indicators THEN turn both off

  dialog.holder <- self
  IF dialog wants a pointer THEN show it and stamp the shown-at time

  IF a level is loaded THEN
    player <- the current view entity, if it is the player
    IF player EXISTS THEN
      IF dialog blocks movement THEN player.stop_all_movement()
      deliver a key-release for aim and for fire to the player     # see note
```

**Invariants** — the synthetic key releases are not tidiness. The player may be holding the
trigger or the aim button when the screen opens; the key-up event will be consumed by the
screen, and without this the weapon stays firing behind the menu until the key is pressed
and released again. Any rebuild that routes input through a modal stack has this bug
waiting for it.

`StartDialog` additionally centres the pointer when the screen asks for that, so a screen
opened with a controller starts with the pointer somewhere useful.

`StartStopMenu` toggles: it closes the screen if it is shown, opens it otherwise.

## `StopMenu` / `StopDialog`

**Contract** — closes a screen. Requires that it is shown. The heads-up state is restored
**only if the screen being closed is the top of the stack**; a screen closed from the
middle merely leaves the stack, and its saved state is inherited by the entry above it.
Removes the screen from the draw list, unbinds it, and hides the pointer if the new top
does not want one.

```text
FUNCTION StopMenu(dialog)
  REQUIRE dialog is shown
  IF dialog IS the top of the stack THEN
    restore the crosshair and heads-up display from the top entry's saved state
    pop the stack
  ELSE
    remove dialog from the middle of the stack, passing its saved state up to the
      entry above it
  remove_from_draw_list(dialog)
  dialog.holder <- none
  IF the new top does not want a pointer THEN hide it
```

**Invariants** — passing the saved state upward is what keeps the restoration correct when
screens close out of order. The entry above inherits the responsibility of restoring what
the removed entry had hidden.

## `SetMainInputReceiver`

**Contract** — the stack's only mutator. With a window and no removal flag, it pushes.
With no window, it pops the top. With a window and the removal flag, it finds that window
somewhere in the stack, copies its saved heads-up state to the entry above it, and removes
it. Asking to make the current top the top again does nothing.

**Invariants** — the "copy upward" step is only safe because the early return has already
excluded the case where the named window *is* the top; there is always an entry above it.
A rebuild must preserve that guard or bound the index.

## `AddDialogToRender` / `RemoveDialogToRender` / `DoRenderDialogs`

**Contract** — draw-list membership and the draw itself.

Adding refuses duplicates in either list, then appends to the pending list if a walk is in
progress and to the live list otherwise, and shows the window. Removing finds the entry,
hides and disables the window, and marks the entry — it does not erase. Drawing walks the
list and draws every entry that is both enabled and shown, in list order, which is
therefore the back-to-front order.

**Invariants** — the draw list's order is its paint order and nothing re-sorts it except
the end-of-frame compaction, which only moves disabled entries. A window's depth is
therefore the order it was opened in.

## `OnFrame`

**Contract** — the per-frame update, and the point at which all deferred mutation is
applied.

```text
FUNCTION OnFrame()
  in_update <- true
  UpdateCursorVisibility()

  top <- the top input receiver
  IF top EXISTS AND top is enabled THEN top.Update()
  FOR EACH entry IN draw_list
    IF entry.enabled AND entry.window is enabled THEN entry.window.Update()

  IF this holder is foremost THEN focus_manager.update(top)

  in_update <- false
  append pending_draws to draw_list ; clear pending_draws
  sort draw_list so that enabled entries come first
  pop trailing disabled entries
```

**Invariants** — the compaction happens strictly after the walk and after the pending merge,
so a window opened *and* closed within one frame is added and then removed without ever
being drawn.

**Notes** — the top receiver is updated once explicitly and then again as part of the draw
list, since it is in both. The source shows this was once an either/or and the branch was
removed; as it stands the modal screen's update runs twice per frame. A rebuild should
update each window once.

## `UpdateCursorVisibility`

**Contract** — shows the pointer whenever the top receiver wants one, and hides it after an
inactivity timeout when nothing does. Only the foremost holder touches the pointer, and
never on a dedicated server, which has none.

**Invariants** — showing is immediate; hiding is delayed by a configurable timeout measured
from the moment the pointer became visible or last moved. The asymmetry is for gamepad use:
the pointer appears the instant it is needed, and lingers so that it does not flicker out
between stick movements.

## `OnExternalHideIndicators`

**Contract** — declares that the heads-up display was hidden by something outside this
holder, by clearing the saved state on *every* stack entry. The effect is that no screen
closing later will turn the display back on. It is how a script or a cutscene takes
ownership of the display away from the screen stack.

## The input entry points

**Contract** — one per event kind: pointer motion, wheel, key press, key release, key hold,
text input, and the three controller equivalents. Every one of them follows the same chain,
and the chain is the file's real contract:

```text
FUNCTION handle(event) -> consumed
  top <- the top input receiver
  IF top IS none THEN RETURN not_consumed          # no screen: the game gets everything
  IF top does not currently process input THEN RETURN not_consumed

  IF top handles the event as a window event THEN RETURN consumed

  IF the event can move focus AND the pointer is visible THEN
    move the focus to the nearest focusable widget in that direction
    RETURN consumed

  IF top does NOT block movement AND a level is loaded THEN
    entity <- the currently controlled entity
    IF entity accepts input THEN
      deliver the event to it as a bound game action
      RETURN not_consumed                          # see note
  RETURN consumed
```

**Invariants**

- Returning "not consumed" after delivering the event to the game is deliberate: the caller
  above this holder must also see the event, because more than one system observes game
  actions. The return value means "was this event *absorbed by the interface*", not "was it
  handled".
- The fall-through is gated by the top window's own declaration of whether it blocks
  movement. That single flag is what distinguishes a screen you can walk behind from one
  you cannot.
- Pointer buttons are translated into window events at the pointer's position *before*
  being offered as keyboard actions, so a click always means a click on whatever is under
  the pointer.
- **Quick-use hotkeys are never passed through.** Using a consumable from a hotkey while a
  screen is open would change the inventory the screen is displaying, behind its back. They
  are the only actions filtered out by name.

## Focus navigation

**Contract** — directional movement of the keyboard/controller focus between widgets, used
when there is no pointer to aim with. From the currently focused widget's centre — or from
the pointer's position if nothing is focused — it asks the focus manager for the nearest
focusable widget in the requested direction and moves the focus there.

Eight directions are distinguished for a stick (the four axes and the four diagonals) and
four for a key binding. A stick must be pushed nearly to its limit before it counts as a
direction, so that a lightly held stick does not skate the focus across the screen.

**Notes** — the focus manager returns two candidates and the first non-empty one is taken.
The second is a fallback for the case where nothing lies strictly in the requested
direction — typically the widget that wraps around. A rebuild wanting predictable
navigation should define that fallback explicitly rather than leaving it to the search.

## Controller pointer emulation

**Contract** — the secondary stick drives a virtual pointer when the top window blocks
movement. Its speed is not proportional to the stick: an *intensity* accumulates while the
stick is held near its limit and decays while it is held short of it, clamped between a
configured minimum and maximum, and is reset the moment the stick is released.

**Invariants** — the intensity is the acceleration curve that makes a stick usable as a
pointer at all: a proportional mapping is either too slow to cross the screen or too fast
to hit a button. It is scaled by the real (not simulation) frame time, so pointer speed is
unaffected by the game being paused — which it usually is while a screen is open.

**Notes** — the intensity is kept in a variable shared by every holder rather than per
holder or per stick. With one pointer on screen this is harmless and it is still the wrong
home for it.

## `CleanInternals`

**Contract** — empties both collections and hides the pointer, without notifying any
window. For teardown, when the windows themselves are about to go.

## `FillDebugTree` / `FillDebugInfo` / `GetDebugType`

**Contract** — debug-only. Renders the input stack and the draw list, each window
contributing its own subtree, into the developer overlay, and exposes the pointer timer and
the foremost flag for inspection. Compiled out of a release build.
