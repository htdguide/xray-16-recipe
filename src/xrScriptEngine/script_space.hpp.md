# src/xrScriptEngine/script_space.hpp

> The single place the guest interpreter and its binding layer enter this module.

**Needs** — [`xrScriptEngine.hpp`](xrScriptEngine.hpp.md) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)

**Used by** — [`ResourceManager_Scripting.cpp`](../Layers/xrRender/ResourceManager_Scripting.cpp.md) · [`BindingsDumper.cpp`](BindingsDumper.cpp.md) · [`Functor.hpp`](Functor.hpp.md) · [`pch.hpp`](pch.hpp.md) · [`script_space_forward.hpp`](script_space_forward.hpp.md) · [`xrScriptEngine.cpp`](xrScriptEngine.cpp.md)

**Tier floor** — T3: it names a dependency set.

## Purpose

One file admits the interpreter and the binding layer; everything else in the module reaches
them through it. That is worth stating rather than transcribing, because the *set* it admits is
the module's actual demand on
[Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer), and a
rebuild shopping for a replacement needs exactly this list:

- class registration with bases, constructors, methods, properties and operators;
- the untyped script value and the table/namespace access built on it;
- iteration over a script table;
- operator export, so a bound vector can be added from script;
- and four *policy* annotations that change how an argument or result crosses the boundary:
  - **adopt** — the guest takes ownership of an object the engine created;
  - **return reference to** — the result's lifetime is tied to one of the arguments, so the
    guest must not treat it as independent;
  - **out value** — a parameter the engine writes through becomes an extra returned value, since
    the guest has no notion of writing through a parameter;
  - **iterator** — a pair of bounds becomes something the guest can iterate.

Those four are the ones a hand-written replacement most often lacks, and every one of them is
visible in shipped script: a script that iterates an engine collection, or receives two values
from a call whose native signature returns one, is depending on them.

## State

Stateless.

## Exported units

- `luabind_it_distance` — the element count of a script table; see
  [`xrScriptEngine.cpp`](xrScriptEngine.cpp.md).

## Notes

The bulk of the file is suppression of compiler complaints from the guest's headers. Incidental:
it survives as the observation that the binding layer is third-party code the engine does not
hold to its own standards.
