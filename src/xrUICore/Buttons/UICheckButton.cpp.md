# src/xrUICore/Buttons/UICheckButton.cpp

> A latching button whose checked state is its press state, which notifies set and reset separately, drives a dependent control's enabled flag, and reads and writes one boolean console variable.

**Needs** — [`UICheckButton.h`](UICheckButton.h.md) · [`UI3tButton.h`](UI3tButton.h.md) · [`Options/UIOptionsItem.h`](../Options/UIOptionsItem.h.md) · [`Hint/UIHint.h`](../Hint/UIHint.h.md) · [`UIMessages.h`](../UIMessages.h.md)
**Used by** — [`UICheckButton.h`](UICheckButton.h.md)
**Tier floor** — T3.

## Purpose

Two unrelated jobs meet here because the shipped options screens want one widget. One is the
toggle behaviour; the other is the settings-item protocol — a check box in an options screen
must be able to say what it was before the player touched it, and be restored to it.

## State

```text
RECORD CheckButton EXTENDS ThreeTexButton, OptionsItem
  backup_value   : bool          # the checked state when the screen opened
  depend_control : optional<Window>
  # checked is not stored: checked == (press_state == pushed)
```

**Invariants** — switch mode is on from construction, so a release never resets the press
state; the only thing that changes the tick is the press handler below or a direct set.

## `on_mouse_down`

**Contract** — on a left press, flip the tick and announce the new state with a dedicated
message — set or reset — and then, unconditionally for any button, also announce a plain
click. Always consumes the press.

```text
FUNCTION on_mouse_down(button) -> bool
  IF button == left
    IF press_state == normal
      press_state <- pushed
      message_target.send(self, CHECK_BUTTON_SET)
    ELSE
      press_state <- normal
      message_target.send(self, CHECK_BUTTON_RESET)
  message_target.send(self, BUTTON_CLICKED)
  RETURN true
```

**Notes** — the plain click notification fires even for a right or middle press, which is
almost certainly unintended but is observable: screens that listen only for clicks will see
one on any press over a check box. Sending three distinct messages for one action is
deliberate, though — a screen can subscribe to "turned on" without having to ask the box what
it now is.

## `on_mouse_action`

**Contract** — bypasses the inherited button state machine entirely and runs the bare window
dispatch, so that the press handler above is the only thing that changes the tick. Without
this bypass the inherited machine would also toggle on the release.

## `update`

**Contract** — per-frame; runs the inherited four-state visual update, then forces the
dependent control's enabled flag to match the tick. The dependent control is not asked
whether it wants to be enabled — it is overwritten every frame, so nothing else may own that
flag.

## `init_check_button`

**Contract** — builds the four-state background from a texture base name, then moves the
label right by the width of the tick graphic and sizes the label's box to the widget width by
the tick's height. The effect is a tick on the left and a label beside it, vertically sized
to the graphic rather than to the widget.

**Notes** — the offset is read from the *enabled* state's texture rectangle, so all four state
textures must be the same size or the label will jump. That constraint is not checked
anywhere and is satisfied by the shipped art.

## The settings-item operations

**Contract** — `set-current` reads the bound console variable as a boolean and sets the tick;
`save-backup` copies the tick into the backup field; `save` writes the tick back through the
console and runs the inherited restart-flag bookkeeping; `undo` restores the tick from the
backup; `is-changed` compares the two. The group machinery in
[`UIOptionsManager`](../Options/UIOptionsManager.cpp.md) calls these across every control on
a page at once.
