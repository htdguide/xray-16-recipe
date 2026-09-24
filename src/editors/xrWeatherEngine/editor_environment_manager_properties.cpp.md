# src/editors/xrWeatherEngine/editor_environment_manager_properties.cpp

> A demonstration file, excluded from the build: one grid row of every kind the property surface supports.

**Needs** — [`Include/editor/property_holder_base.hpp`](../../Include/editor/property_holder_base.hpp.md) · [`Include/editor/ide.hpp`](../../Include/editor/ide.hpp.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: it holds throwaway values behind accessors and builds grid rows from them.

## Purpose

Not part of the program. The file's own first line says it is excluded from the build, and
its code targets an older shape of the editor interface than the one that ships — it
constructs the weather system's root object with a body that does nothing but populate a
test grid.

It survives in the tree as **a worked example of the property surface**, and that is the
only reason to keep it: it exercises, in one place, every row type
[`property_holder_base`](../../Include/editor/property_holder_base.hpp.md) offers, which
no shipping file does.

## State

A handful of file-scope values — an integer, a string, a flag, a colour, a real — each
wrapped in a tiny holder with a reader and a writer, so the grid has something live to
bind to. Nothing reads them back.

## What it demonstrates

```text
# one row per supported shape, all in one property page
add_property(holder)                                    # a nested page
add_property(integer)                                   # free integer
add_property(integer, min, max)                         # bounded integer
add_property(integer, names[])                          # integer chosen from a name list
add_property(integer, (value,name) pairs[])             # integer chosen from an enumeration
add_property(text, extension, mask, folder, caption)    # text chosen by file browser
add_property(text, names[])                             # text chosen from a list
add_property(bool)                                      # a checkbox
add_property(bool, [false_label, true_label])           # a two-value named choice
add_property(colour)
add_property(real)                                      # free real
add_property(real, min, max)                            # bounded real
add_property(real, (value,name) pairs[])                # real chosen from an enumeration
```

**Notes** — Three things are worth taking from it into a rebuild.

Every row is offered in two forms — bound to a reader/writer pair, or bound to a field
directly. The second form is not a shortcut: a field binding means the grid writes the
value with no chance to react, and every place in this module that must recompute
something on change (rebuild a shader, re-resolve an ambient) uses the first form for
exactly that reason.

An integer row can be *named* two ways: by a list of labels indexed by value, or by an
explicit value-to-label mapping. The first is for dense small ranges, the second for
sparse or non-contiguous ones.

A text row bound to a file browser carries the extension, the file mask, the starting
folder and the dialog caption together, because a browsable path is not a string with a
picker bolted on — the four travel as one unit everywhere in this module.

The folder constant left in the example is an absolute path on a build machine that no
longer exists. Shipping code builds that argument from the virtual filesystem instead; see
[`editor_environment_detail.cpp`](editor_environment_detail.cpp.md).
