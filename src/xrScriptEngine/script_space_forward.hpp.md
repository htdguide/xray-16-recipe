# src/xrScriptEngine/script_space_forward.hpp

> Names the three binding-layer types an engine header may mention without admitting the binding
> layer itself.

**Needs** — [`script_space.hpp`](script_space.hpp.md)

**Used by** — [`script_engine.hpp`](script_engine.hpp.md)

**Tier floor** — T4: three names.

## Purpose

An engine header that merely *mentions* a script value — a field holding a script callback, a
method returning a namespace — should not force the binding layer's headers on every file that
includes it. This declares the three names that mention costs nothing: the untyped script value,
the typed callable of [`Functor.hpp`](Functor.hpp.md), and the checked conversion from a script
value to an engine type.

The decision it records is that **the binding layer is contained**: only files that manipulate
the value stack pay for it. In a rebuild with a module system this file does not exist, but the
containment it enforces should.
