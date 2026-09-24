# src/Include/editor/interfaces.hpp

> The two exported symbols of the editor library: bring the editor up, and tear it down.

**Needs** — [`ide.hpp`](ide.hpp.md) · [`engine.hpp`](engine.hpp.md) · [Seam: Threads, atomics and process services](../../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — [`ide.hpp`](ide.hpp.md) · [`entry_point.cpp`](../../editors/xrWeatherEditor/entry_point.cpp.md)
**Tier floor** — T1: it names the calling convention and signature of two symbols looked up by name from a dynamically loaded library.

## Purpose

The weather editor is a separate application built on the same engine core, and it is split in two across a language boundary: an *engine host* that owns the level, the renderer and the weather data, and an *editor library* that owns the windows, the property grid and the timeline. The host loads the library at run time and finds exactly two symbols in it.

This file declares those two signatures. It is three lines of substance and it is the whole contract of the dynamic load.

## State

`Stateless.`

## `initialize` / `finalize`

```text
FUNCTION initialize(out ide : Ide)
FUNCTION finalize(inout ide : Ide)
```

**Contract** — `initialize` constructs the editor's root object and writes it into the caller's slot; `finalize` destroys whatever is in the slot and clears it. Both take the slot by reference rather than returning or taking a value, so the host's single variable is the one place the editor's identity lives and neither side can hold a stale copy.

**Invariants** — exactly one editor exists at a time; `initialize` asserts it is not being called twice. The host's own engine implementation must be constructed *before* `initialize`, because the editor's root object takes a reference to it during construction.

**Notes** — Two things about this file are worth carrying forward and one is not.

The **out-parameter idiom** is load-bearing across a dynamically loaded boundary: it keeps allocation and deallocation on the library's side (the library constructs, the library destroys) while the host holds the only reference. A rebuild in a language with a module system expresses this as a factory function and a disposal, and the property that must survive is that **the module that allocated the editor is the module that frees it**.

The **implementation language differs across the boundary**: the host is unmanaged native code, the editor library is a managed one. That is why this handshake is so narrow — every type that crosses it must be expressible on both sides, which rules out the language's own containers, strings and exceptions. It is also why [`property_holder_base.hpp`](property_holder_base.hpp.md) declares its own two small records with an explicit layout.

What does not survive: the file records, in a comment, an earlier signature in which `initialize` also received the engine and a calling convention was spelled out explicitly. Neither is needed now — the engine is reached through a global on the host side, and the convention is the platform default.
