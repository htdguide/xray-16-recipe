# src/editors/xrWeatherEditor/property_editor_file_name.cpp

> Turns a file the artist picked into a path relative to the asset root, which is the only form the document may hold.

**Needs** — [`property_editor_file_name.hpp`](property_editor_file_name.hpp.md) · [`property_file_name_value.hpp`](property_file_name_value.hpp.md) · [`property_file_name_value_base.hpp`](property_file_name_value_base.hpp.md) · [`property_container.hpp`](property_container.hpp.md) · [Seam: Windowing and input](../../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — reached through its declarations in [`property_editor_file_name.hpp`](property_editor_file_name.hpp.md); callers name that, not this file.
**Tier floor** — T3: pure presentation; text crosses to the row through its own interface

## Purpose

The modal chooser behind every asset-reference row. Its substance is not the dialog — it is the translation in both directions between an absolute path the platform deals in and the root-relative, extension-stripped, lowercase reference the weather document stores. Get that translation wrong and the file the editor writes no longer resolves in the game.

## State

```text
RECORD FileChooser
  dialog : platform file dialog     # one instance, reused across every row and every open
```

The dialog is configured once at construction with the settings that never vary per row: single selection only, the file must exist and its path must exist, shortcuts are followed to their target, names are validated, the working directory is restored afterwards, and no extension is appended by the dialog itself. Everything that *does* vary per row — filter, title, starting directory, assumed extension — is applied at each open from the row's own chooser settings.

**Notes** — The dialog appending extensions is switched off deliberately. This file decides what happens to the extension, based on the row's stripping setting; a dialog that also had opinions would apply them first and invisibly.

Restoring the working directory matters because the engine half of the process resolves its own relative paths against it. A dialog that left the working directory wherever the artist last browsed would change how the *engine* resolves files, from inside a file picker.

## `GetEditStyle`

**Contract** — Modal whenever there is a row context; otherwise the grid's default.

## `EditValue`

**Contract** — Opens the chooser on the row's asset root, seeded with the row's current value; on confirmation, converts the chosen absolute path into a stored reference and writes it through the row's adapter. Blocks. Does nothing on cancel. Returns the value it was handed, unchanged.

```text
FUNCTION EditValue(context, services)
  IF no context OR no services OR no window service THEN RETURN default

  adapter = context.subject.property(context.row)     # AS file-naming row
  root    = absolute(lowercase(adapter.initial_directory()))
  ext     = lowercase(adapter.default_extension())

  configure dialog with adapter.filter(), adapter.title(), ext
  IF current value is non-empty THEN
    dialog.start_at(root + current value)             # seed the exact file
  ELSE
    dialog.start_in(root)

  IF dialog.show() != confirmed THEN RETURN default

  chosen = absolute(lowercase(dialog.chosen_path))
  IF chosen begins with root THEN
    chosen = chosen without that prefix               # make it root-relative
  IF adapter.remove_extension() AND chosen ends with ext THEN
    chosen = chosen without that suffix
  adapter.SetValue(chosen)
  RETURN default
```

**Invariants** — The stored value is lowercase, relative to the row's asset root, and carries its extension only if the row says it should. Every reference in the weather document obeys this; the engine's resolver assumes it.

**Notes** — Three details in that sequence are decisions and not plumbing.

*Lowercase everywhere.* Both the root and the chosen path are lowercased before they are compared or stored, because the asset references in the document are matched case-insensitively by an engine that runs on a case-insensitive filesystem. Without it the same texture picked from a differently-cased directory produces a reference that will not match on a case-sensitive one. Lowercasing the *whole absolute path* rather than the relative part is what makes the prefix comparison reliable.

*Absolute-then-strip rather than relative arithmetic.* Both sides are resolved to absolute form first, then compared as a prefix. A path that does not start with the root is stored absolute — the artist picked a file from outside the asset tree, and the reference will not resolve. The editor does not object. That is a real gap: the one case worth rejecting is the one that silently produces a broken document.

*Seeding by concatenation.* The dialog is seeded with root plus the stored value as a full file name rather than by setting a starting directory, so it opens with the current asset selected and not merely with its folder listed — the difference between "change this texture" and "find a texture" as an authoring gesture. The starting-directory form is used only when the row is empty.

An alternative chooser with thumbnail browsing is present but compiled out. Choosing a sky texture by name is a poor authoring experience and the original knew it; nothing depends on the alternative and a rebuild is free to do better.
