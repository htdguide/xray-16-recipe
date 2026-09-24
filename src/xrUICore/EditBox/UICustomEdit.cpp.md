# src/xrUICore/EditBox/UICustomEdit.cpp

> Routes keyboard and text input into a shared line editor while focused, computes the window of the string that fits in the box around the caret, and draws a caret glyph at the measured offset.

**Needs** — [`UICustomEdit.h`](UICustomEdit.h.md) · [`Lines/UILines.h`](../Lines/UILines.h.md) · [`Static/UIStatic.h`](../Static/UIStatic.h.md) · [`ui_focus.h`](../ui_focus.h.md) · [`UIMessages.h`](../UIMessages.h.md) · [`xrEngine/line_edit_control.h`](../../xrEngine/line_edit_control.h.md) · [Seam: Windowing and input](../../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — [`UICustomEdit.h`](UICustomEdit.h.md)
**Tier floor** — T2.

## Purpose

Three jobs: deciding when this field owns the keyboard, feeding the shared editor, and
solving the horizontal scroll problem — a string longer than the box must be shown as a
window that always contains the caret.

## State

```text
RECORD Edit EXTENDS Static
  editor        : LineEditor           # owns the string, the caret and the filter
  has_keyboard  : bool
  visible_slice : text (at most 256 characters)   # what is actually drawn
  caret_offset  : real (pixels)        # measured width of the text before the caret
  needs_relayout: bool
  read_only     : bool
  next_in_tab_chain : optional<Edit>
```

**Invariants**

- The widget's text control is the *slice*, not the string. Anything asking the widget for
  its text gets the editor's full string instead; the two are deliberately different.
- The text control is configured once, at construction, into the only mode that makes sense
  for an entry field: simple (unwrapped), no colour markup, vertically centred. An edit box
  therefore never wraps and never honours inline colour tags.
- Keyboard capture is a property of the *parent*, not of this widget: the widget asks its
  parent to route keys to it, and the parent evicts whoever held the routing before.

## Construction and `init`

**Contract** — creates the shared line editor at 256 characters, registers the four bound
keys, selects the input filter, clears the text and starts without the keyboard. The filter
choice is a ladder: read-only mode additionally puts the editor in permanently-selected mode
(so the content can be copied but not changed); otherwise number-only, filename-mode, or
unrestricted.

**Notes** — registering the widget as focusable is what lets gamepad and keyboard
directional navigation land on it.

## `on_mouse_action`

**Contract** — a left press or a left double-click over an unfocused field takes the keyboard.
Never consumes the action, so the press also reaches the widget's other behaviours and its
parent.

## `on_keyboard_action` / `on_text_input`

**Contract** — while focused, forward press, hold and release to the editor and consume; while
unfocused, consume nothing. Composed text — the platform's text-input channel, separate from
key events — is likewise forwarded only while focused. The split between the two channels is
what makes non-Latin input work, and a rebuild must keep them separate.

## `update` / `draw`

**Contract** — `update` steps the editor's own per-frame work (key repeat, caret blink
timing). `draw` recomputes the visible slice whenever the editor reports a change, then draws
the static normally and, if focused, queues an underscore at the caret's measured offset.

```text
FUNCTION recompute_visible_slice()
  before <- editor.text_before_caret()

  # slide the left edge right until the text before the caret fits
  left <- 0
  WHILE measure(before[left..]) > box_width AND left < length(before)
    left <- left + 1

  # then grow the slice rightwards while it still fits
  n <- 1
  WHILE measure(editor.text[left .. left+n]) < box_width
        AND n < length(editor.text) - left
    n <- n + 1
  visible_slice <- editor.text[left .. left+n]

  caret_offset <- measure(IF password_mode THEN '*' repeated length(before[left..])
                                           ELSE before[left..])
```

**Notes** — both loops re-measure the whole candidate string on every step, so the cost is
quadratic in the string length. At 256 characters and once per change that is invisible, but
a rebuild should measure incrementally.

The caret is drawn as an underscore glyph from the same font, positioned by measuring the
slice before it — not by a character-cell calculation — which is why it lands correctly in a
proportional font. It is drawn every frame while focused, with no blink.

In password mode the caret offset is measured against a run of asterisks rather than the real
characters, because that is what is drawn.

## The four bound keys

**Contract** —

- *escape*: if the field has content, clear it (unless read-only). If it is already empty,
  release the keyboard and announce `EDIT_TEXT_CANCEL`. So escape is "clear, then cancel", two
  presses, not one.
- *return* (and the numeric-pad return): release the keyboard and announce `EDIT_TEXT_COMMIT`.
- *backquote*: bound to nothing, explicitly. This shadows the console's open key so that
  typing a backquote into a field does not drop the console over it.
- *tab*: commit as above, then move the keyboard to the next field in the chain and focus it.
  Does nothing when no next field was set.

## `send_message`

**Contract** — when this field is told the keyboard was taken from it while it believed it had
it, mark itself unfocused and announce a commit. Losing the keyboard is therefore equivalent
to pressing return, never to pressing escape — clicking away from a field keeps what was
typed.

## `capture_focus`

**Contract** — asks the parent to route the keyboard here or stop, tells the editor to start
or stop consuming raw input, and records the state. The two halves must happen together: the
editor also hooks the input layer directly, for key repeat.

## `enable`

**Contract** — disabling the field announces a keyboard-capture-lost to the message target,
which drives the commit path above. A field that is disabled while being typed into commits.
