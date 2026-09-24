# src/xrScriptEngine/script_callback_ex.h

> A bound script function the engine can call like a native one, which reports its own failures
> and unbinds itself rather than letting one propagate into the simulation loop.

**Needs** — [`script_engine.hpp`](script_engine.hpp.md) · [`Functor.hpp`](Functor.hpp.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)

**Used by** — [`client_spawn_manager.h`](../xrGame/client_spawn_manager.h.md) · [`enemy_manager.h`](../xrGame/enemy_manager.h.md) · [`entity_alive.cpp`](../xrGame/entity_alive.cpp.md) · [`patrol_path_manager.h`](../xrGame/patrol_path_manager.h.md)

**Tier floor** — T2: the decision it encodes is what a failure crossing the language boundary
costs. The failure-handling shape differs between build configurations, which is the reason it
is a type and not a call.

## Purpose

Most of the engine's script callbacks are optional: an entity may or may not have a death
handler, a UI element may or may not have a click handler. This is the type those slots hold. It
answers three questions the call sites need answered the same way everywhere — is anything
bound, does the call carry a receiver object, and what happens when the script fails.

It is a header with no implementation file because every instantiation is per-signature; the
decisions are all here.

## State

```text
RECORD ScriptCallback<R>
  target   : optional<script_value>   # the function to call
  receiver : optional<script_value>   # passed as the first argument when present
```

**Invariants**

- The two are bound together or not at all: setting a target clears any previous receiver.
- Testing the callback for emptiness tests the *target*. A callback with a receiver and no
  target is not callable and never occurs.
- Comparing two callbacks compares both halves, and two unset values compare equal — so "is this
  the handler already installed" is answerable without special-casing the unset state. This
  matters because the engine re-binds handlers on every entity spawn and must not stack
  duplicates.

## `set` / `clear`

**Contract** — `set` binds a function, optionally with a receiver, discarding whatever was
bound. `clear` unbinds both and leaves the slot in the same state a freshly constructed one is
in. Neither touches the VM beyond taking or releasing a reference.

**Notes** — The original clears by destroying and reconstructing the held values in place, which
is a C++ answer to "release the VM reference deterministically without an assignable empty
value". A rebuild assigns "none" and is done.

## Invocation

**Contract** — Calls the bound function with the given arguments, prepending the receiver when
there is one. Returns the script's result converted to the callback's result type. When nothing
is bound it returns a zero-valued result and does nothing. Never propagates a failure out of the
call.

```text
FUNCTION invoke(args...) -> R
  IF target IS none
    RETURN zero_value_of(R)
  TRY
    IF receiver IS present
      RETURN call(target, receiver, args...)
    RETURN call(target, args...)
  ON script failure
    report the failure through the engine's error path
  ON any other failure
    clear()                 # a callback that fails this way is unbound and never called again
  RETURN zero_value_of(R)
```

**Invariants**

- *The engine's frame is never taken down by a script callback.* This is the point of the type.
  A handler that errors logs and returns a zero-valued result; the entity carries on.
- *A callback that fails in an unrecognised way unbinds itself.* The distinction is between a
  script error, which is expected and reportable, and a failure the engine cannot name — a
  destroyed VM, a corrupted binding — where calling again would repeat the same failure every
  frame for the rest of the session. Unbinding turns an unbounded repeated failure into a single
  one.

**Notes**

- *Build configuration is load-bearing here.* When the binding layer is compiled with
  exceptions, a script error arrives as a distinguishable failure and is reported precisely.
  When it is compiled without them — the shipping configuration — the layer's own error path has
  already logged and terminated before this code is reached, so the reporting branch is dead and
  the only reachable handler is the unbinding one. A rebuild gets to choose *one* of these, and
  the choice is visible to modders: with exceptions, a broken script handler is survivable; in
  the shipped build it is not. See
  [`script_engine.cpp`](script_engine.cpp.md) for where that is decided.
- *The zero-valued result* has the same hazard as everywhere else in this module: for a result
  type whose zero means something, a failed callback is indistinguishable from a callback that
  returned that value. An optional result is the better shape and changes no shipped script.
- The comparison helper that treats two unset values as equal exists because the binding layer's
  own equality on an unset value is not defined; that is a property of the library, not a
  decision.
