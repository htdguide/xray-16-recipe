# src/xrUICore/Buttons/UIButton.cpp

> The press state machine that turns a stream of mouse actions into exactly one click notification, plus the accelerator and hover-hint behaviour every derived button inherits.

**Needs** — [`UIButton.h`](UIButton.h.md) · [`UIBtnHint.h`](UIBtnHint.h.md) · [`Static/UIStatic.h`](../Static/UIStatic.h.md) · [`Cursor/UICursor.h`](../Cursor/UICursor.h.md) · [`Windows/UIWindow.h`](../Windows/UIWindow.h.md) · [`ui_focus.h`](../ui_focus.h.md) · [Seam: Windowing and input](../../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — [`UIButton.h`](UIButton.h.md)
**Tier floor** — T3: a three-state machine over an event stream; the only awkward part is that it must read the *current* physical mouse-button state, which is a platform query.

## Purpose

Every interactive control in the toolkit is a button underneath — tabs, check boxes, radio
buttons, spin arrows, scroll-bar arrows, list items. This file decides what "a click" means,
and the answer is deliberately not "a press": a press that drifts off the widget and
releases elsewhere is not a click, and a press that drifts off and comes *back* is.

## State

```text
RECORD Button EXTENDS Static
  press_state   : ENUM { normal, pushed, released_outside }
  is_switch     : bool          # latching: a release does not reset the state
  accelerators  : list<Accelerator>   # exactly 4 slots, fixed
  hint_text     : text

RECORD Accelerator
  code     : int      # a scancode when is_key, otherwise a named game action
  is_key   : bool
  # invariant: code == -1 means the slot is empty and is skipped
```

**Invariants**

- `press_state` is `pushed` only while the pointer button that started the press is still
  down, *or* while the button is a latching switch. The one exception is repaired in the
  focus-lost handler below.
- `cursor_over_window` — inherited from the window — is the authority on whether the pointer
  is inside; the state machine never re-tests the rect itself.

## `on_mouse_action`

**Contract** — consumes one mouse action and advances the state machine; returns whether the
action was consumed. Children are offered the action first (the inherited window walk), so a
button with children behaves as a container when the pointer is over a child. Fires
`BUTTON_DOWN` on the transition into `pushed` and `BUTTON_CLICKED` — via `on_click` — on a
release that happens while the pointer is still inside.

```text
FUNCTION on_mouse_action(x, y, action) -> bool
  IF inherited.on_mouse_action(x, y, action) RETURN true

  MATCH press_state
    normal:
      IF action IN { left_down, left_double_click }
        press_state <- pushed
        message_target.send(self, BUTTON_DOWN)
        RETURN true
    pushed:
      IF action == left_up
        IF cursor_over_window THEN on_click()
        IF NOT is_switch THEN press_state <- normal
      ELSE IF action == mouse_move AND NOT cursor_over_window AND NOT is_switch
        press_state <- released_outside      # armed, but a release here must not click
    released_outside:
      IF action == mouse_move AND cursor_over_window
        press_state <- pushed                # came back: re-arm
      ELSE IF action == left_up
        press_state <- normal
  RETURN false
```

**Notes** — a double-click is admitted as a press because a fast second click must still
depress the button; the double-click itself is separately reported by the window layer to
whoever wants it.

## `on_click`

**Contract** — sends `BUTTON_CLICKED` to the message target. Overridden by derived buttons to
add a sound or a second notification; it is the single funnel every click path goes through,
including the accelerator path, which is why the accelerator does not duplicate the press
machine.

## `draw_texture`

**Contract** — draws the button's own texture, offset one unit right and one unit down while
the button is depressed. When the button is set to stretch, the quad is the widget rect;
otherwise it is the texture's own pixel size. Honours the static's rotation angle.

**Notes** — the one-unit push offset is in the toolkit's virtual 1024×768 space, so it scales
with the screen. It is the entire visual feedback for a plain button; the three-texture
button replaces it with a per-state texture instead.

## `draw_text`

**Contract** — draws the inherited text, then — if this button currently owns the shared
hover hint — marks the hint as wanted for this frame. The hint is not drawn here: it is
drawn after everything else so that it is never covered by a later sibling.

**Notes** — the push offset is computed here and then not applied. The text does not shift
when the button depresses; only the texture does. Reproducing that asymmetry is the faithful
choice, and it is almost certainly an oversight in the original rather than a decision.

## `update`

**Contract** — per-frame. When the pointer has rested on this button for longer than the
dwell delay, no other widget currently owns the shared hint, and this button has hint text,
claim the hint, fill it with this button's text, and place it near the cursor.

```text
FUNCTION update()
  inherited.update()
  IF NOT cursor_over_window OR hint_text IS empty OR hint.owner EXISTS
    RETURN
  IF now <= focus_receive_time + 700 * time_scale
    RETURN                      # 700 ms of stillness before a hint appears

  hint.claim(self, hint_text)
  # place the hint box so it fits on screen: try above-right of the cursor,
  # then above-left, then below-left, then below-right pushed clear of the
  # cursor glyph
  r <- rect(cursor, hint.size)
  r <- move_up(r);            IF fits(screen, r) THEN done
  r <- move_left(r);          IF fits(screen, r) THEN done
  r <- move_down(r);          IF fits(screen, r) THEN done
  r <- move_right_and_down(r, 45)
  hint.position <- r.top_left
```

**Notes** — the dwell is scaled by the simulation's time factor, so hints appear slower in
slow motion; that is consistent with the rest of the UI's timers and is cheap to reproduce.
The 45-unit drop in the last fallback clears the cursor image, whose drawn height is about
40 units.

## `on_focus_lost`

**Contract** — when the pointer leaves the button: if the button is depressed, not latching,
and the physical mouse button is *no longer* held, force the state back to normal, and
release the hint if this button owns it.

**Notes** — this is the repair for a press that ended while the pointer was over a different
widget, where no release action ever reaches this button. It is the one place the toolkit
asks the platform for the live button state instead of relying on the event stream, and a
rebuild needs that query to exist. The original carries a `???` comment here; the condition
is in fact inverted from what the name suggests — it fires when the key *is* reported down —
which is recorded here as behaviour to copy, not as a decision to defend.

## `on_keyboard_action`

**Contract** — on a key press, if the key matches any accelerator slot, fire `on_click` and
consume the key; otherwise pass it down to children. A button reacts to its accelerator
whether or not it has focus and whether or not the pointer is anywhere near it — the
accelerator is a screen-wide shortcut owned by the widget.

## `set_accelerator` / `get_accelerator` / `is_accelerator`

**Contract** — write, read and test one of four accelerator slots. An out-of-range slot index
is refused rather than trusted. A slot marked `is_key` matches a raw scancode; a slot not so
marked holds a named game action and matches if the pressed key is bound to that action in
either the gameplay binding table or the UI binding table.

```text
FUNCTION is_accelerator(key) -> bool
  FOR EACH slot IN accelerators
    IF slot.code == -1 CONTINUE
    IF slot.is_key
      IF slot.code == key RETURN true
    ELSE
      IF bound(slot.code, key, gameplay_context) RETURN true
      IF bound(slot.code, key, ui_context)       RETURN true
  RETURN false
```

**Notes** — checking both binding contexts is what lets one accelerator serve "the key the
player bound to *cancel* in game" and "the key the UI reserves for *back*" at once. Scancodes
are the storage unit for bindings throughout the engine, so a rebuild's input seam must
deliver layout-independent scancodes, not characters.
