# src/xrUICore/MessageBox — the modal dialog

> A dialog whose entire child set is chosen by a style name in the data, and which reports the
> player's answer as a distinct message per button.

Part of [chapter 15](../README.md).

## What this directory is responsible for

The "are you sure", "connection failed", "enter a password" box. The interesting decision is
that the *shape* of the dialog is data: a style name selects which children exist — which
buttons, whether there is an edit field — and the control wires the standard accelerators onto
whichever buttons that style produced.

## The load-bearing ideas

**The style is the dialog's type.** Yes/no, ok, ok/cancel, question-with-text and so on are not
subclasses; they are named styles read from the layout, each naming a child set. Adding a
dialog shape is a data change.

**Every answer is announced twice.** A button's click becomes a style-specific answer message
sent once naming the button and once naming the dialog. The first form lets an owner
distinguish which button; the second lets an owner that only cares that the dialog closed bind
one handler. Both are needed because the shipped screens use both.

**Accelerators are bound to whatever exists.** The confirm and cancel keys are attached after
the child set is built, to whichever buttons the style produced, so a style with no cancel
button simply has no cancel key.

**The body text goes through the localization table**, and reading it back returns the resolved
string, not the key.

## The twins

| Twin | Role |
|---|---|
| [`UIMessageBox.cpp`](UIMessageBox.cpp.md) | Building the child set from the named style, wiring accelerators onto whichever buttons exist, and the doubled answer messages |
| [`UIMessageBox.h`](UIMessageBox.h.md) | Its declaration and the style vocabulary |
