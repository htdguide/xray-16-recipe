# src/xrCommon/xr_unordered_map.h

> The engine's hashed key-to-value table, allocating through the engine allocator.

**Needs** — [`xr_allocator.h`](xr_allocator.h.md) · [`xrCore/xrstring.h`](../xrCore/xrstring.h.md)
**Used by** — [`_stl_extensions.h`](../xrCore/_stl_extensions.h.md) · [`ScriptExporter.cpp`](../xrScriptEngine/ScriptExporter.cpp.md)
**Tier floor** — T3: an allocator injection point, plus the hashing decision it defers.

## Purpose

Exists to inject an allocator into the hashed table. It is the newest of the container
aliases and the only one with no declaration shorthands, which is a hint that it arrived
after the macro habit was abandoned rather than any statement about the type.

## `xr_unordered_map`

**Contract** — a table mapping keys to values with expected-constant lookup, parameterised
by a hash of the key and an equality on the key. Defaults are the key type's own hash and
its own equality. Iteration order is unspecified.

**Invariants**

```text
# Hash and equality must agree: equal keys must hash equally.
#   This is where the interned-string type earns its design. Its equality is
#   identity of the interned record, and its hash is the checksum the intern table
#   already computed and stored, so the pair agrees by construction and hashing a
#   key of any length costs one field read. See xrCore/xrstring.h.

# Iteration order is not observable output.
#   No caller may write a hashed table's iteration to a file or to the wire. Where
#   ordered output is needed the ordered table is used instead - which is why both
#   exist.
```
