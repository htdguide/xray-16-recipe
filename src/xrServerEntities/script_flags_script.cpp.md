# src/xrServerEntities/script_flags_script.cpp

> Exports the three bit-field widths to scripts, since entity records store most of their booleans as packed flags.

**Needs** — [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer) · [`xrCore/_flags.h`](../xrCore/_flags.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3.

## Purpose

Almost every entity record in this chapter keeps its boolean state as a packed bit field —
the spawn flags, the state flags, the game-mode mask — and scripts read and write those
fields directly. This file publishes the 8-, 16- and 32-bit flag types under the names
`flags8`, `flags16` and `flags32`, with identical surfaces.

## The exported surface

- `get` / `zero` / `one` — read the whole word, clear it, set every bit.
- `assign` — replace the word, from another flag word or a raw value.
- `invert` — flip every bit, flip against another word, flip a mask.
- `or` / `and` — combine with a mask, or with another word masked.
- `set(mask, value)` — set or clear the bits of a mask.
- `is(mask)` — every bit of the mask is set. `is_any(mask)` — at least one is.
  `test(mask)` — the mask's bits, as a boolean.
- `equal(other)` / `equal(other, mask)` — compare whole or masked.

**Invariants** — the predicate operations answer a genuine boolean across the boundary. The
underlying type answers an integer, and each predicate is wrapped for exactly that reason:
a script comparing the raw result against `true` would otherwise fail for every non-zero
value that is not exactly one.

**Notes**

**`one` is wrapped for the narrow widths and not for the wide one.** The underlying
"set every bit" operation is written in terms of the 32-bit value, so applying it to an
8- or 16-bit word sets bits that do not exist and then truncates unpredictably. The wrapper
assigns all-bits-set at the word's own width instead. The 32-bit type uses the native
operation directly because there is nothing to widen. This is a real correctness fix hiding
in what looks like boilerplate.

**`or` and `and` are keywords in many languages and are exported under those names anyway,**
because shipped scripts call them that. Lua permits it as a table key; a rebuild whose
script language does not will have to alias them and accept that shipped scripts break —
which makes this one of the small places where conformance criterion 10 constrains the
choice of script language beyond "Lua 5.1".
