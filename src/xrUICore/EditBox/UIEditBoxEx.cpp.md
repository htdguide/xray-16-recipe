# src/xrUICore/EditBox/UIEditBoxEx.cpp

> An entry field that owns a nine-slice frame outright and keeps it in step with its own rect.

**Needs** — [`UIEditBoxEx.h`](UIEditBoxEx.h.md) · [`UICustomEdit.h`](UICustomEdit.h.md) · [`Windows/UIFrameWindow.h`](../Windows/UIFrameWindow.h.md)
**Used by** — [`UIEditBoxEx.h`](UIEditBoxEx.h.md)
**Tier floor** — T3.

## Purpose

Composition only.

## Construction and teardown

**Contract** — creates the frame child, attaches it, and switches the text control into complex
mode. The frame is *not* marked auto-delete and is released explicitly.

**Notes** — this is the one place in the toolkit where a child is attached but not auto-deleted
and is then deleted by the owner. The window base already detaches and releases auto-delete
children; here the explicit release happens first, leaving the parent's child list holding a
released pointer until the base destructor runs. Reproduce the *intent* — the frame is owned
by the field — and let the rebuild's ownership model make the order safe.

## `init_custom_edit`

**Contract** — sizes the frame to the requested size at the child origin, then sets the
field's own rect.

## `init_texture` / `init_texture_ex`

**Contract** — forward straight to the frame. Unlike the sibling class there is no lazy
creation, because the frame always exists.
