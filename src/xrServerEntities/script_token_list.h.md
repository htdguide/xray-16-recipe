# src/xrServerEntities/script_token_list.h

> A script-built vocabulary of name/number pairs, for configuration reads constrained to a named set.

**Needs** — [`script_token_list.cpp`](script_token_list.cpp.md) · [`xrCore/xr_token.h`](../xrCore/xr_token.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`script_ini_file.h`](script_ini_file.h.md) · [`script_properties_list_helper_script.cpp`](script_properties_list_helper_script.cpp.md) · [`script_token_list.cpp`](script_token_list.cpp.md) · [`script_token_list_script.cpp`](script_token_list_script.cpp.md)
**Tier floor** — T2: it must produce the exact terminated-array shape the configuration reader consumes.

## Purpose

Declares the surface; the substance is in
[`script_token_list.cpp`](script_token_list.cpp.md). A **token** is a (name, number) pair:
the configuration file holds the name, the record holds the number. This type lets a script
assemble such a vocabulary at run time and hand it to a configuration read or to the
editor's property model, so that a mod can introduce a new enumerated field without an
engine change.

Its sibling [`script_rtoken_list.h`](script_rtoken_list.h.md) is the form where the number
is the position. Here the number is given explicitly, which is what makes this form safe for
values that reach a save: entries can be reordered, added or removed without renumbering the
rest.

## the exported units

- the pair type itself — a name and a number;
- `add`, pairing a name with a number;
- `remove`, by name;
- `clear`;
- `id`, name to number;
- `name`, number to name;
- access to the underlying array for the native consumer.
