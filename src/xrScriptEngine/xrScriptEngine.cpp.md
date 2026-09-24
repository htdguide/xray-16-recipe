# src/xrScriptEngine/xrScriptEngine.cpp

> Counts the elements of a binding-layer iteration, on this side of the module boundary.

**Needs** — [`xrScriptEngine.hpp`](xrScriptEngine.hpp.md) · [`script_space.hpp`](script_space.hpp.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)

**Used by** — nothing in this recipe; entry point or dead code.

**Tier floor** — T3: one linear count.

## Purpose

Walks a binding-layer iterator to its end and reports how many steps that took — the size of a
script table, as seen from the engine.

It is a separate exported function, and this file exists at all, only because callers in other
modules must not instantiate the iteration themselves: doing so would pull the binding layer's
headers into every consumer and, in a separately-linked build, give each of them its own copy of
the iterator's internals. A rebuild with a uniform module model deletes the file and asks the
table for its size.

## State

Stateless.

## `luabind_it_distance`

**Contract** — Steps an iterator until it equals the end and returns the number of steps. Linear;
no allocation; the iterator is consumed.

**Notes** — Linear rather than constant because a script table has no cheap length: the count
that matters here includes non-integer keys, which the interpreter's own length operator does
not report.
