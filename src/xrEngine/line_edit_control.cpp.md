# src/xrEngine/line_edit_control.cpp

> A single-line text editor over a fixed buffer: selection, word motion, clipboard, undo of one edit, and key auto-repeat driven by the frame clock.

**Needs** — [`line_edit_control.h`](line_edit_control.h.md) · [`edit_actions.h`](edit_actions.h.md) · [`xr_input.h`](xr_input.h.md) · [`device.h`](device.h.md) · [Seam: Windowing and input](../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — [`line_edit_control.h`](line_edit_control.h.md)
**Tier floor** — T1: a fixed byte buffer edited in place, with four derived slices published as raw spans for the renderer to draw.

## Purpose

Every place the engine asks a player to type — the console, the interface's text fields —
uses this. It is an editor over one fixed-size byte buffer, with the whole of the editing
vocabulary a person expects: a caret, a selection anchored at a second position, word-wise
motion and deletion, insert versus overwrite, clipboard, and a one-step undo.

It does not draw. What it publishes instead is **four slices of the buffer** — before the
caret, before the selection, the selection, after the selection — recomputed on every
change, so a renderer can draw the line in three runs with the middle one highlighted
without understanding any of this.

It also owns a decision the platform does not make for it: **auto-repeat is generated here**,
from the engine's own clock, with an acceleration curve, rather than taken from the
operating system's key-repeat events. That is the only way the repeat rate can be the same
on every platform and can accelerate the way a game console expects.

## State

```text
RECORD LineEdit
  buffer_size    : int             # fixed at construction, clamped to [8, 4096]
  text           : bytes           # the line, NUL-terminated
  undo           : bytes           # what the last edit replaced — exactly one step
  pending        : bytes           # characters accepted this event, not yet merged
  slice_*        : bytes (4)       # the published slices; derived, never authoritative

  caret          : int             # the insertion point
  anchor         : int             # the other end of the selection
  sel_low        : int             # derived: min(caret, anchor)
  sel_high       : int             # derived: max(caret, anchor)

  actions        : map<scancode, Action>   # the key handler chain; see edit_actions
  key_modifiers  : set of modifier flags   # re-read from the platform every event

  mode           : one of standard | number-only | read-only | file-name
  insert_mode    : bool            # overwrite rather than insert
  selection_off  : bool            # this control never selects (a plain field)
  extend         : bool            # this event should move the anchor with the caret

  caret_visible  : bool            # the blink phase
  blink_time     : real
  repeat_time    : real            # time since the last synthetic repeat
  accel          : real            # repeat acceleration, grows while a key is held
  held_time      : real            # how long the current key has been down
  repeating      : bool
  last_frame     : int
  changed_frame  : int             # used to answer "did this change recently"
```

**Invariants**

- `caret` is always within the text; every operation that can move it re-clamps.
- `sel_low <= sel_high`, and both equal `caret` when the control does not select.
- The four slices are recomputed after every operation that can change text, caret or
  selection, and are never read as anything but output.
- A tab or newline never reaches the buffer: both are replaced by a space at insertion. This
  is a one-line control and a pasted paragraph must not break it.

## Key handling — a chain per key

**Contract** — each scancode owns a *chain* of handlers, each guarded by a required modifier
set. Pressing a key walks its chain, most-recently-registered first, and the first handler
whose modifiers are satisfied runs and stops the walk. Registering a new handler for a key
pushes it onto the front of that key's chain rather than replacing it.

```text
FUNCTION assign(scancode, required_modifiers, handler)
  new_link = a callback link guarding handler with required_modifiers
  new_link.next = the key's current chain
  actions[scancode] = new_link
```

**Notes** — the chain is what lets one key carry several meanings by modifier without a
lookup table of combinations: `Delete` alone deletes forward, `Ctrl+Delete` deletes a word,
`Shift+Delete` cuts, and each was registered independently. A "required modifier set" of
*none* matches unconditionally, so the unmodified handler must be registered first to sit at
the end of the chain — which the initialization order arranges.

Six modifier keys additionally carry a link whose only job is to *record* that the modifier
is down before continuing along the chain, so a chord's state is set even if the modifier's
own press arrives as a separate event. See [`edit_actions.cpp`](edit_actions.cpp.md).

The chain is stored per scancode in a table sized to the platform's full key count. Several
scancodes share handler objects through the chain, which is why teardown must deduplicate
before releasing — an incidental consequence of the sharing, not a decision.

## The four modes

**Contract** — the mode chosen at initialization decides both which keys are bound and which
characters are admitted.

```text
standard    every editing key; every character except the control set
read-only   only selection, copy and motion are bound; no key can change the text
number-only admits digits and the sign characters only
file-name   rejects the characters a path may not contain:  ' " \ / < > ? | ; : @ # $ % ^ & * =
```

**Notes** — read-only is enforced by *binding fewer keys*, not by a check at the mutation
sites. That is fragile — text input bypasses the key chain entirely — and in fact **a
read-only control still accepts typed characters**, because `on_text_input` filters by mode
through `char_is_allowed`, which has no read-only case. A rebuild should make the mode a
single gate at the mutation boundary.

The file-name character set is the union of what the shipped platforms forbid, not any one
platform's. That is the right choice for a game whose save files must be copyable between
machines.

## `on_key_press`

**Contract** — the event path for a key. Normalizes state, clears the pending characters,
runs the key's chain, merges whatever the chain produced, then re-derives the selection and
the slices. Re-reads the modifier state from the platform at the end rather than trusting the
event.

```text
FUNCTION on_key_press(control, scancode)
  IF scancode is out of range THEN RETURN
  IF this is not a synthetic repeat
    held_time = 0 ; accel = 1
  extend = true
  clamp the caret; clear pending; recompute the selection bounds

  run the chain for scancode                       # may edit, move, or fill pending

  IF scancode is a control modifier
    extend = false                                 # see note
  re-terminate the buffer; clamp the caret
  merge pending into the text at the selection

  IF extend AND (shift is not held OR something was inserted)
    anchor = caret                                 # collapse the selection
  recompute the selection bounds

  repeating = false ; repeat_time = 0
  re-read the modifier state; rebuild the slices
```

**Notes** — the anchor rule at the end is the whole selection model in one line. The anchor
follows the caret — that is, the selection collapses — unless Shift is held, in which case
the caret moves and the anchor stays, growing the selection. And it collapses *anyway* if
characters were inserted, because typing over a selection replaces it and there is nothing
left to select.

The control-modifier exception exists because Ctrl is itself a key press, and without it
merely pressing Ctrl before a chord would collapse the selection the chord was meant to act
on.

`extend` is set true at the top of every press and cleared only in that one case, which
reads as a flag but is really "this event is a caret movement unless told otherwise".

## `on_text_input`

**Contract** — the *other* input path, and the one that actually produces characters. The
platform's composed-text events arrive here as a UTF-8 string; it is converted to the
engine's single-byte encoding under the system locale, filtered character by character
against the mode, and inserted. Always collapses the selection afterwards.

**Notes** — key events and text events are deliberately separate (see the windowing seam):
a key event carries a physical key, a text event carries what the keyboard layout and any
composition produced. Editing commands come from the first; characters come from the second.
That is what makes the control work with a non-Latin layout, a compose key or an input
method — none of which the key path could handle.

The conversion to a single-byte encoding under the system locale is where the engine's
non-UTF-8 text model (system requirements §4) bites: a character the current codepage cannot
represent is lost here, silently. A rebuild holding text as Unicode throughout deletes this
step and the loss.

## `on_frame` and auto-repeat

**Contract** — advances three timers from the real, never-paused clock: the caret blink, the
repeat interval, and how long the current key has been held. Sets the repeat flag when the
interval elapses and accelerates it.

```text
FUNCTION on_frame(control)
  re-read the modifier state
  dt = clamp(real ms since last frame, 0, 66) in seconds     # see note

  blink_time = blink_time + dt
  caret_visible = blink_time <= 0.3
  IF blink_time > 0.4 THEN blink_time = 0                    # a 0.4 s cycle, 75% on

  repeat_time = repeat_time + dt * accel
  IF repeat_time > repeat_interval                            # tunable, default 0.15 s
    repeat_time = 0 ; repeating = true ; accel = accel + 0.2
  held_time = held_time + dt

  IF nothing changed for more than one frame
    clear the "recently changed" flag

FUNCTION on_key_hold(control, scancode)
  ignore the modifiers and tab entirely
  IF repeating AND held_time > 5 * repeat_interval            # the initial delay
    re-enter on_key_press as a synthetic repeat
```

**Notes** — four numbers, and each is a decision.

**The delta is clamped at 66 ms.** A stalled frame must not fire five repeats at once; the
caret must not jump across the line because the level was loading.

**The initial delay is five repeat intervals, the repeat interval itself is tunable
(`g_console_sensitive`, default 0.15 s).** One knob controls both, which keeps the ratio
fixed: the delay before repeat begins is always five times the interval between repeats,
which is roughly what an operating system does.

**Acceleration adds 0.2 to a multiplier on every repeat**, so a held arrow key crosses a long
line at a usable speed while a short tap still moves one character. It is reset on any
release and on any non-repeat press.

**The clock is the never-pausing real-time one.** A player typing into the console while the
game is paused must still get a blinking caret and key repeat.

Modifiers and tab are excluded from repeat: repeating a modifier means nothing, and
repeating tab would cycle the console's completion list uncontrollably.

## Editing operations

**Contract** — each is small; the ones with a decision inside are listed.

```text
move_home / move_end            caret to 0 / to the end
move_left / move_right          one character; left stops at 0, right is clamped after
move_left_word                  skip spaces leftwards, then run back to the previous
                                terminator, then step forward off it
move_right_word                 run forward to the next terminator, then skip spaces
delete_back / delete_forward    delete the selection; if empty, one character
delete_word_back / _forward     force the extend flag on, move by a word, delete, restore
select_all                      anchor to 0, caret to the end
flip_insert_mode                overwrite instead of insert
copy / cut / paste              the platform clipboard
undo                            re-insert what the last edit replaced
```

**Notes** — **word-wise deletion is implemented as "select a word, then delete the
selection"**, with the Shift state saved and forced around it. Reusing the motion and the
deletion instead of writing a third algorithm is the right call; saving and restoring the
*real* Shift state around it is what keeps the player's actual modifier state intact.

The **terminator set** is what defines a word: whitespace, brackets, quotes, and the
arithmetic and punctuation characters — the underscore among them. So `wpn_fire` is three
words to this control, which is arguably wrong for a console of underscore-separated names
and is what a rebuilder will notice first.

**Undo is exactly one edit deep**, and its buffer holds *what was replaced*, not the line
before. Undo re-inserts that text at the current selection, which means undo after moving
the caret inserts the old text in the new place. It is a crude mechanism that is right in
the common case (type, realize, undo) and surprising otherwise.

**Insert mode consumes one character of the tail** on each insertion and on each redraw, via
a one-byte adjustment threaded through both the merge and the slice computation. That is
also what makes the caret in overwrite mode *cover* the character it would replace rather
than sit before it.

**Paste writes straight into the pending buffer** and lets the normal merge insert it, so a
paste replaces a selection and respects the buffer limit with no special case.

## `add_inserted_text` — the merge

**Contract** — the one function that changes the text. Replaces the selection with the
pending characters, truncating if the result would exceed the buffer, and leaves the caret
after the insertion. Records what it replaced as the undo buffer.

```text
FUNCTION merge(control)
  IF pending is empty THEN RETURN
  replace every tab and newline already in the text with a space      # see purpose
  undo = text[sel_low .. sel_high]
  truncate pending so that sel_low + length(pending) fits the buffer
  text = text[0 .. sel_low] + pending + text[sel_high + overwrite_adjust .. end]
  IF the result fits
    commit it AND caret = sel_low + length(pending)
  clamp the caret
```

**Notes** — the result is built in a scratch buffer and committed only if it fits, so an
over-long paste leaves the line untouched rather than half-applied. The truncation applies
to the *pasted text*, not the line, which is the behaviour a person expects.

## `update_bufs` — the published slices

**Contract** — recomputes the four slices from the text, the caret and the selection bounds,
and stamps the frame so a reader can ask whether anything changed recently.

**Notes** — publishing slices rather than a caret index is what keeps the renderer ignorant
of the editor: it draws three runs and a caret between two of them. The cost is four buffer
copies on every keystroke, which at typing speed is free and at machine speed would not be.

## `on_ir_capture` / `on_ir_release`

**Contract** — turn the platform's text-input mode on when this control takes input focus and
off when it loses it. That mode is what makes composed-character events arrive at all, and
on a touch platform it is what raises the on-screen keyboard.

## `SwitchKL`

**Contract** — advance to the next keyboard layout, **only** when the engine has exclusively
grabbed the keyboard. When it has not, the operating system's own layout switch works and
this must not double-fire.

**Notes** — bound to Ctrl+Shift and Alt+Shift, the two conventional layout-switch chords.
The whole function exists because grabbing the keyboard for a game takes the layout switch
away from the player, and a Cyrillic-layout player typing into the console needs it back.
Implemented on one platform only; elsewhere the grab does not take the switch away.

## `remove_spaces` / `split_cmd`

**Contract** — two free helpers the console uses on a line before executing it. The first
collapses runs of spaces and strips the leading and trailing ones, in place. The second
splits a line at the first space into a command and the rest, tolerating a line with no
space.

**Notes** — these live here rather than with the console because they operate on the same
fixed-buffer text model. They are string utilities and a rebuild will have them already.
