# src/xrScriptEngine/script_callStack.cpp

> The call stack as the editor sees it: which frame is selected, and where that frame's source
> line is.

**Needs** — [`script_callStack.hpp`](script_callStack.hpp.md) · [`script_debugger.hpp`](script_debugger.hpp.md)

**Used by** — [`script_callStack.hpp`](script_callStack.hpp.md)

**Tier floor** — T3: a list of file-and-line pairs with a cursor.

## Purpose

The debugger's variable pane and its source view both follow "the selected frame". This owns
that selection and the per-frame source positions needed to act on it.

## State

```text
RECORD CallStack
  frames   : list<(file : text, line : int)>   # outermost first, as sent to the editor
  selected : int                               # -1 when nothing is selected
```

**Invariants** — `selected` is either -1 or a valid index. Clearing resets it to -1, which is the
state between a resume and the next stop.

## Contract

**`Clear`** — Drops every frame and deselects. Called at the start of each stop.

**`Add`** — Appends one frame's file and line. The frame's display description is passed but not
retained: the editor renders the description, and the engine only needs to be able to navigate.

**`SetStackTraceLevel`** — Selects a frame without navigating. Used to reset to the innermost
frame at the start of a stop.

**`GotoStackTraceLevel`** — Selects a frame *and* tells the editor to open that file at that
line. An out-of-range index is ignored rather than failing, because the index comes from another
process and may refer to a stack that has already moved on.

**Notes** — Two parallel lists in the original (one of lines, one of paths) with a third that is
cleared but never filled; that third is dead. A rebuild uses one list of pairs, which is what
the record above says.
