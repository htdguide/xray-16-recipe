# src/xrUICore/EditBox/UIEditBox.cpp

> Lazily creates a stretched-line border sized to the field, and maps the field's content onto a string console variable with backup and undo.

**Needs** — [`UIEditBox.h`](UIEditBox.h.md) · [`UICustomEdit.h`](UICustomEdit.h.md) · [`Windows/UIFrameLineWnd.h`](../Windows/UIFrameLineWnd.h.md) · [`Options/UIOptionsItem.h`](../Options/UIOptionsItem.h.md)
**Used by** — [`UIEditBox.h`](UIEditBox.h.md)
**Tier floor** — T3.

## Purpose

A thin composition file. The only thing worth stating is the border's lifecycle and the
null-backup convention in the settings operations.

## `init_texture` / `init_texture_ex`

**Contract** — creates the border child on the first call, attaches it with auto-delete, loads
the named texture into it, and sizes it to the field's current rect at the child origin.
Subsequent calls reload the texture into the existing border. The short form supplies the
default UI shader.

**Invariants** — the border is sized from the field's size *at load time*, so a field resized
after its texture is loaded must re-run the geometry call below.

## `init_custom_edit`

**Contract** — places the border at the child origin at the requested size if it exists, then
sets the field's own rect. The border is not created here — a field with no texture simply has
none.

## The settings-item operations

**Contract** — `set-current` loads the console variable's string into the field; `save` writes
the field's string back through the console; `save-backup` snapshots the current string;
`undo` restores the snapshot; `is-changed` compares them.

```text
FUNCTION effective_backup() -> text
  # an empty snapshot means "never taken"; fall back to the live console value
  RETURN IF backup IS empty THEN console_value(entry) ELSE backup
```

**Notes** — treating an absent snapshot as "read the console instead" means a field that was
never backed up reports itself unchanged, which is what stops an options page from writing
every field it contains when the player changes one. The same convention appears in no other
settings control; the rest snapshot eagerly.
