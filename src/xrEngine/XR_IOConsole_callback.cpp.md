# src/xrEngine/XR_IOConsole_callback.cpp

> What the arrow keys and the tab key mean inside the console's edit field, and how the edit buffer is rewritten under the cursor.

**Needs** — [`XR_IOConsole.h`](XR_IOConsole.h.md) · [`xr_ioc_cmd.h`](xr_ioc_cmd.h.md) · [Seam: Debug overlay UI](../../SYSTEM-REQUIREMENTS.md#seam-debug-overlay-ui)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: key dispatch and buffer replacement

## Purpose

The edit field is the overlay toolkit's, not the engine's. The toolkit owns the text, the
caret and the selection; it calls back when it sees a completion key or a history key, and
the callback is the console's chance to rewrite the buffer. This file is that callback and
the six navigation actions it dispatches to.

The design consequence worth carrying into a rebuild: the console never simulates typing.
It replaces the whole buffer in one act and tells the field to reload, which is why every
path here does the same three things — clear the caret, clear the length, insert the new
text.

## State

`Stateless.` It reads and writes the edit buffer and the two cursors declared in
[`XR_IOConsole.h`](XR_IOConsole.h.md).

## The callback

**Contract** — invoked by the edit field for two event kinds: a completion request (the tab
key) and a history request (the up and down arrows). Returns without complaint for anything
else. Always clears the field's selection afterwards, so the rewritten text is not left
highlighted and destroyed by the next keystroke.

```text
FUNCTION on_edit_event(console, field)
  IF event IS completion
    (cmd, completed) = find_next_command(edit_buffer)
    IF cmd EXISTS AND completed is non-empty
      replace the field's whole contents WITH completed
    clear the field's selection
    RETURN

  IF event IS history
    ctrl = the control modifier is held
    alt  = the alt modifier is held

    IF ctrl AND alt
      up -> jump to the first suggestion;  down -> jump to the last
    ELSE IF alt
      up -> page up through suggestions;   down -> page down
    ELSE IF ctrl
      up -> previous command;              down -> next command
      replace the field's whole contents WITH edit_buffer
    ELSE
      up -> previous suggestion;           down -> next suggestion
      replace the field's whole contents WITH edit_buffer
    clear the field's selection
```

**Notes** — the modifier layering is the decision: bare arrows move through *suggestions*,
control-arrows move through *history*, alt-arrows page, and both together jump to the ends.
Suggestions get the unmodified keys because they are what a person is looking at while
typing.

The two branches that rewrite the field do so because the suggestion and history movers
write into the console's own buffer; the field must then be told to reload from it. The
jump and page branches do not rewrite, because they only move the highlight.

The source also carries a disabled branch for shift-tab, which would complete *backwards*.
It is disabled because the toolkit does not deliver a callback for shift-tab, not because
the behaviour was unwanted. A rebuild whose edit field reports it should restore it: the
logic is a lower-bound lookup followed by one step back.

## `Prev_tip` / `Next_tip`

**Contract** — the bare arrow keys. They move through suggestions *unless* there is nothing
to suggest — an empty edit buffer, or suggestions the person has dismissed — in which case
they fall through to the command history instead. That fallback is what makes the up arrow
on an empty line recall the last command, which is the single most used key in the console.

```text
FUNCTION previous_tip(console)
  IF edit_buffer is empty OR suggestions are suppressed
    older_history(); select_command()      # fall through to history
    RETURN
  select_previous_tip()
```

## `Prev_cmd` / `Next_cmd`

**Contract** — move the history cursor and copy the entry it lands on into the edit buffer.
Always paired, because the cursor is invalid until moved (see
[`XR_IOConsole_control.cpp`](XR_IOConsole_control.cpp.md)).

## `Begin_tips` / `End_tips` / `PageUp_tips` / `PageDown_tips`

**Contract** — jump to the first or last suggestion, or move by one visible page. Each
re-clamps afterwards so the visible window follows. The page size is the number of visible
rows, so a page move lands the row that was just off-screen at the edge rather than skipping
past it.

**Notes** — jumping to the end sets the window's first row directly rather than letting the
clamp compute it, then clamps; that is redundant but harmless, and it is the only place the
window index is written outside the clamps.
