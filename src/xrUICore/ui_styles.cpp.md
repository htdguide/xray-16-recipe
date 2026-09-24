# src/xrUICore/ui_styles.cpp

> Discovers the installed UI styles as subdirectories of the configuration tree and switches between them by re-pointing the two global path prefixes every UI document lookup goes through.

**Needs** — [`ui_styles.h`](ui_styles.h.md) · [`xrCore/XML/XMLDocument.hpp`](../xrCore/XML/XMLDocument.hpp.md) · [`ui_base.h`](ui_base.h.md)
**Used by** — [`ui_styles.h`](ui_styles.h.md)
**Tier floor** — T3: directory enumeration and string assembly.

## Purpose

A "style" is a directory of replacement layout and texture-description documents. Switching
style does not touch a single widget: it changes where document lookups *start*, so every
subsequent load finds the style's copy if there is one and the default otherwise. The whole
mechanism is two global path prefixes and a reload.

This is a project-local feature, not something the shipped games need — but it is the
mechanism by which a total conversion replaces the interface, and it constrains the document
loader, which must try the style path before the default path on every open.

## State

```text
RECORD StyleManager
  tokens   : list<(name : text, id : int)>   # id 0 is always the built-in default
  style_id : int                              # currently selected
```

Two globals outside this record, owned by the document layer but written here:

```text
UI_PATH                  : text   # logical root of the active style's documents
UI_PATH_WITH_DELIMITER   : text   # the same with a trailing separator
```

**Invariants**

- Style id 0 is the default and its path prefix is the shared constant `"ui"`, never an
  allocated string. Every other style's prefix is an allocated string of the form
  `ui/styles/<name>`, and it must be released before it is replaced or the manager dies.
  That asymmetry — one borrowed constant among allocated strings — is the source of every
  ownership special case in this file.
- The token list is terminated by a sentinel entry with no name, because it is also consumed
  by the console-variable machinery, which expects that shape.
- A style's identifiers are assigned by enumeration order at startup. They are therefore
  **not stable** across installations: a saved settings file that stores the numeric id would
  select the wrong style if a directory were added. The name is the durable key.

## `UIStyleManager` construction

**Contract** — seeds the token list with the built-in default, then enumerates the
subdirectories of the active style root and appends one token per directory, numbering from
one. Finishes with the sentinel. Reads directories only; loads nothing.

```text
FUNCTION construct(mgr)
  mgr.tokens = [ ("ui_style_default", 0) ]
  next_id = 1
  FOR EACH directory IN "<UI_PATH>/styles/" (top level only)
    strip the trailing separator from the name
    mgr.tokens append (copy of name, next_id); next_id = next_id + 1
  mgr.tokens append sentinel
```

**Notes** — the enumeration runs against the *currently active* style root, which at startup
is the default. Styles are therefore discovered under the default tree; a style cannot itself
contain nested styles.

## `SetupStyle`

**Contract** — selects a style by identifier and rewrites the two global path prefixes.
Selecting the already-selected style does nothing. Selecting the default restores the shared
constants and releases the previously allocated prefixes; selecting any other builds
`ui/styles/<name>` and its delimited twin. Does not reload anything.

```text
FUNCTION setup_style(mgr, id)
  IF id == mgr.style_id THEN RETURN
  IF id == DEFAULT
    release the allocated prefixes if the current style is not the default
    UI_PATH = "ui"; UI_PATH_WITH_DELIMITER = "ui/"
  mgr.style_id = id
  IF id == DEFAULT THEN RETURN
  name = name of `id` in mgr.tokens
  UI_PATH = "ui/styles/" + name
  UI_PATH_WITH_DELIMITER = UI_PATH + "/"
```

**Notes** — the new prefix is built from the *default* root, not from the current one, so
switching from one non-default style to another does not nest.

## `SetStyle`

**Contract** — selects a style by name, optionally triggering a reload. Returns whether the
name was found. This is the entry point the console, the debug overlay and scripts use;
`SetupStyle` is the lower half.

## `Reset`

**Contract** — raises the engine's UI-reset notification, which makes every subscriber throw
away and reload everything it loaded from data — the texture registry, the colour
definitions, and every open screen. This is what makes a style change visible; without it the
new prefixes would only affect documents opened later.

## `GetCurrentStyleName` / `GetCurrentStyleId` / `DefaultStyleIsSet` / `GetToken`

**Contract** — accessors. The name lookup is a linear scan of the token list and asserts if
the current identifier is not in it, which can only happen if the list were rebuilt without
resetting the selection. `GetToken` hands out the whole list for the console and the debug
overlay to present as a choice.

## `~UIStyleManager`

**Contract** — releases every allocated token name (the default's is a literal and must be
left alone) and, if a non-default style is active, the two allocated path prefixes. The
globals are left pointing at freed memory, which is tolerable only because this runs at
shutdown.

**Notes** — the manual string ownership here is entirely incidental to the original language.
A rebuild holds owned strings in the token list and assigns owned strings to the prefixes,
and this destructor disappears along with every special case in `SetupStyle`.
