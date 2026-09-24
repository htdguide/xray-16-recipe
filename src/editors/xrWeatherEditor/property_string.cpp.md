# src/editors/xrWeatherEditor/property_string.cpp

> One grid row bound to a text value in the engine, with a copy made at every crossing in both directions.

**Needs** — [`property_string.hpp`](property_string.hpp.md) · [`property_holder_include.hpp`](property_holder_include.hpp.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: transfers text between two runtimes' representations, each allocation freed by the side that made it

## Purpose

The text counterpart of [`property_integer.cpp`](property_integer.cpp.md), and the base of every text adapter in this directory. Its substance is not the binding — that is the same getter/setter pair as everywhere else — but the ownership rule for text crossing between the two halves of the process.

## State

```text
RECORD TextProperty
  getter : callback() -> native text    # native; owned — a private copy
  setter : callback(native text)        # native; owned
```

## `construct(getter, setter)` · `release`

**Contract** — Takes private copies of the callbacks; frees them exactly once. Release rule as in [`property_float.cpp`](property_float.cpp.md).

## `GetValue`

**Contract** — Calls the getter and returns a **copy** of the text as the presentation layer's own string. The engine's text is not retained past the return.

**Notes** — The copy is the contract, not an optimisation to remove. Text the engine hands out points into engine-owned storage — often an interned string it may release or reuse — and the grid holds what it is given for as long as the row is visible. Anything short of a copy is a dangling reference the moment the document changes.

## `SetValue`

**Contract** — Converts the grid's string into a native buffer, calls the setter with it, and frees the buffer as soon as the setter returns. The setter must therefore copy anything it intends to keep.

```text
FUNCTION SetValue(text)
  buffer = native_copy_of(text)     # this side allocates
  setter(buffer)                    # engine copies whatever it keeps
  free(buffer)                      # this side frees, unconditionally
```

**Invariants** — Each side frees what it allocated, and neither retains the other's buffer past the call. This single rule governs every text crossing in the editor; the file-name and chooser adapters inherit it unchanged.

**Notes** — The buffer is freed even when the setter did nothing with it, because the alternative — transferring ownership of a buffer across the boundary — makes the freeing rule depend on the engine's behaviour, which the editor cannot see. A rebuild that passes text by value across the boundary makes the whole question disappear, at the cost of a copy the original was avoiding.
