# src/xrUICore/EditBox — text entry

> A static that owns a line editor, holds the keyboard while focused, and scrolls its visible
> window of text to keep the caret in view. Twice more, with different frames.

Part of [chapter 15](../README.md).

## What this directory is responsible for

The **custom edit** is the entry field: it routes keyboard and text input into a shared line
editor while focused, computes which window of the string fits in the box around the caret,
and draws a caret glyph at the measured offset. Its two derivatives differ only in dressing
and in what they bind to — one wears a lazily created three-segment stretched line and maps
its content onto a string console variable with backup and undo; the other owns a nine-slice
frame outright, keeps it in step with its own rectangle, and wraps its text.

## The load-bearing ideas

**Two input channels, not one.** Navigation and editing commands arrive as scancodes; the
characters that end up in the string arrive on the separate composed-text channel. An edit box
consumes from both and must not conflate them — the shipped localizations include non-Latin
input, which only the text channel delivers correctly.

**The line editor is shared, not owned.** The editing logic — insertion, deletion, caret
movement, selection, history — is the engine's console line editor, the same one the developer
console uses. The widget supplies focus, geometry and drawing; it does not reimplement editing.

**The visible window is measured, not counted.** Which substring is shown is decided by
measuring text width in the current font against the box width, anchored so the caret stays
inside. A rebuild that counts characters instead will break on proportional fonts, which all
of these are.

**Keyboard capture is what "focused" means here.** The box takes the keyboard while it has
focus, so the rest of the tree sees no key events — including the accelerators that would
otherwise close the screen.

**The console binding carries backup and undo**, because an edit box in a settings screen must
be revertible when the player cancels the page. That is the settings-control protocol, not an
edit-box feature.

## The twins

| Twin | Role |
|---|---|
| [`UICustomEdit.cpp`](UICustomEdit.cpp.md) | Routing input into the line editor, the measured visible window, and the caret glyph |
| [`UICustomEdit.h`](UICustomEdit.h.md) | The entry field's declaration |
| [`UIEditBox.cpp`](UIEditBox.cpp.md) | The stretched-line border created on demand, and the string console variable with backup and undo |
| [`UIEditBox.h`](UIEditBox.h.md) | Its declaration |
| [`UIEditBoxEx.cpp`](UIEditBoxEx.cpp.md) | The owned nine-slice frame, kept in step with the field's rectangle |
| [`UIEditBoxEx.h`](UIEditBoxEx.h.md) | Its declaration, with wrapping text |
