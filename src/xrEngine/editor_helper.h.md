# src/xrEngine/editor_helper.h

> The adapter layer between the engine's own vocabulary and the debug overlay toolkit — scoped identifiers, engine-typed widgets, and the scancode translation.

**Needs** — [`editor_helper.cpp`](editor_helper.cpp.md) · [`xr_level_controller.h`](xr_level_controller.h.md) · [Seam: Debug overlay UI](../../SYSTEM-REQUIREMENTS.md#seam-debug-overlay-ui) · [Seam: Windowing and input](../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — [`Environment_editor.cpp`](Environment_editor.cpp.md) · [`editor_base.cpp`](editor_base.cpp.md) · [`editor_base_input.cpp`](editor_base_input.cpp.md) · [`editor_helper.cpp`](editor_helper.cpp.md) · [`thunderbolt.cpp`](thunderbolt.cpp.md) · [`xr_efflensflare.cpp`](xr_efflensflare.cpp.md)
**Tier floor** — T3: widget glue and a lookup table.

## Purpose

The debug overlay speaks in raw floats, char buffers and its own key enumeration. The
engine speaks in shared strings, packed colours, typed enumerations and scancodes. Every
debug panel in the engine would otherwise convert at each call site. This file does it once.

It is *substantive* rather than a mere declaration header, because the conversions here are
decisions: how an engine colour becomes an editable one, how an enumeration becomes a
control, and how the engine's input vocabulary maps onto the toolkit's.

## State

Stateless.

## Scoped brackets

Two records exist only to pair a "push" with its matching "pop" at the end of a block: one
for the identifier scope, one for a dropdown's body. Both are pure C++ lifetime mechanics
and survive into a rebuild as *whatever that language uses to make a bracket unforgettable*
— a defer, a with-block, a closure-taking function. The decision they encode is that the
overlay's push/pop pairs are error-prone enough to be worth a construct.

## `ItemHelp`

**Contract** — attaches an explanatory tooltip to the previous widget, optionally preceded
by a dimmed marker glyph on the same line. The tooltip appears after a short hover delay and
wraps at thirty-five character widths of the current font. That wrap width is a readability
choice, not a constraint.

## `MenuItemWithShortcut`

**Contract** — a menu entry that displays the key currently bound to a named game action as
its shortcut text. The binding is read live from the input layer's action table, so a
rebound key shows through with no other change. See
[`xr_level_controller.h`](xr_level_controller.h.md) for the action vocabulary.

## `Selector`

**Contract** — edits an enumerated value by presenting a slider over the index range whose
*displayed text* is the name of the selected item rather than a number, with typing
disabled. Clamps the incoming value into range before drawing, so a configuration file
holding a stale enumerator does not draw out of bounds.

**Notes** — one special case is worth keeping: when the enumeration has exactly two values,
a click that does not turn into a drag toggles instead of doing nothing. A two-value slider
is functionally a checkbox, and a user will click it like one.

## `InputText`

**Contract** — edits an interned string through a fixed-size path-length scratch buffer,
writing back and reporting a change only when the text was edited. The fixed buffer is the
mechanism; the decision is that *these fields are paths and a path has a maximum length*
(see the preface's filesystem assumptions).

## `ColorEdit4`

**Contract** — edits a colour held as one packed 32-bit word through the toolkit's
four-float colour picker, converting in both directions. Reports whether the value changed.

## `xr_key_to_imgui_key`

**Contract** — total function from the engine's input code space (keyboard scancodes and
gamepad buttons in one numbering) to the toolkit's key enumeration; anything unmapped
yields "no key".

**Notes** — the table is long and mostly mechanical, but it records two facts a rebuild
needs. First, the engine's key space is *one* space covering keyboard and gamepad, so a
single translation suffices and the overlay gets gamepad navigation for free. Second, a
large tail of scancodes — media keys, international and keypad exotica, the function keys
past twelve — is deliberately unmapped: the toolkit has no name for them, and dropping them
is correct because the overlay is debug-only.
