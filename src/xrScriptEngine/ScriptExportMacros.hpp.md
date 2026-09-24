# src/xrScriptEngine/ScriptExportMacros.hpp

> The declaration form for a C++ class whose selected methods a Lua table may override,
> with a guaranteed path back to the native implementation.

**Needs** — [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer) · [`script_engine.hpp`](script_engine.hpp.md)

**Used by** — nothing in this recipe; entry point or dead code.

**Tier floor** — T2: it is about dispatch — which of two implementations of a method runs — and
about swallowing a failure in the guest half. Nothing here is layout- or device-facing.

## Purpose

Some engine classes are not merely *visible* to script; they are *subclassable* from script. A
shipped script declares a table deriving from an exported class, defines a handful of methods
on it, and the engine then calls those methods where it would have called its own. This file is
the declaration form for that: it describes, per class, which methods are overridable, and what
happens when script does not override one, or overrides it badly.

It is macros in the original because the same four-line shape is repeated for every arity and
every return kind. In a rebuild it is one mechanism, not two dozen; the arity families here are
pure ceremony and carry no decisions.

## State

Stateless. What it declares is dispatch, not data.

## The overridable-method contract

For each method declared overridable, two things must exist:

```text
# 1. the dispatching implementation, installed in place of the native one
FUNCTION dispatch(self, args...) -> R
  TRY
    RETURN call_script_method(self, "<method name>", args...)
  ON conversion failure
    IF developer build
      log "runtime error: cannot convert result of <method> to <type>"
    RETURN zero_value_of(R)
  ON any other failure
    RETURN zero_value_of(R)

# 2. the escape hatch: the native implementation, reachable by name from script
FUNCTION native(self, args...) -> R
  RETURN the base class's own implementation, non-virtually
```

**Invariants**

- Both halves are registered under the *same* script-visible name. The binding layer chooses:
  when the script object defines the method, the script definition runs; otherwise the native
  one does. A rebuild must keep both reachable — a script subclass that overrides one method of
  five still needs the other four to work.
- The native half dispatches statically, to the class the wrapper wraps. If it dispatched
  virtually it would re-enter the script override and spin.
- `dispatch` never propagates a failure. A script override that errors, returns the wrong type,
  or does not exist yields a zero-valued result and the engine carries on. This is a deliberate
  and load-bearing choice: these methods are called from the simulation loop, often per frame
  per entity, and a modded script must not be able to take the process down from inside one.
  The cost is that a broken override is silent outside a developer build, where the conversion
  failure is at least logged.
- Procedures with no result swallow everything unconditionally — there is nothing to return, so
  there is nothing to report.

**Notes**

- Some methods take their results through out-parameters rather than returning them. The
  declaration form for those passes a reference into script as a mutable object, and the native
  escape hatch dereferences it. The only decision here is that the script side sees a handle it
  mutates, not a value it returns.
- The zero-valued fallback is unsafe for any result type whose zero is meaningful — a
  probability, a distance, an entity handle. The original accepts that; a rebuild with an
  optional result type should return "no answer" instead and let the caller decide, which is
  strictly better and changes no shipped script.
