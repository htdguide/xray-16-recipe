# src/xrScriptEngine/Functor.hpp

> A typed handle to a script function: call it like a native function of a known signature, and
> convert one in and out of script automatically.

**Needs** — [`script_space.hpp`](script_space.hpp.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)

**Used by** — [`script_callback_ex.h`](script_callback_ex.h.md) · [`script_engine.hpp`](script_engine.hpp.md)

**Tier floor** — T2: it is a conversion rule between the engine's type system and the guest's,
and it must decide what a script `nil` means when a function was asked for.

## Purpose

The binding layer gives the engine an untyped script value. Most of the engine wants a *typed
callable*: "a function returning a floating-point number", so that the call site reads like a
call and the result conversion is chosen once rather than at every use. This is that wrapper,
plus the rules for passing one across the boundary in either direction.

It is a header with no implementation file because it is entirely a conversion rule.

## State

```text
RECORD Functor<R>
  target : optional<script_value>    # a script function, or nothing
```

Invariant: a functor holding nothing is not callable, and is the representation of a script
`nil` that arrived where a function was expected.

## `functor<R>`

**Contract** — Holds a script value and calls it with any arguments, converting each argument
into the VM and the result back out as `R`. A functor holding nothing is not callable. Construct
from a script value or empty.

**Notes** — The no-result form is spelled separately in the original because the compiler of its
day objected to returning a result of a type with no values. That is a language artifact and
survives as nothing.

## Conversion in and out of script

**Contract** — Defines how a functor crosses the boundary:

```text
# script -> engine
FUNCTION accepts(value) -> bool
  RETURN value is a function OR value is nil

FUNCTION from_script(value) -> functor<R>
  IF value is nil THEN RETURN an empty functor
  RETURN a functor holding value

# engine -> script
FUNCTION to_script(f)
  push the held value
```

**Invariants** — **Nil is accepted where a function is expected**, and becomes an empty functor
rather than a conversion failure. This is not a convenience: shipped scripts pass nil to clear a
handler, and a conversion failure in the shipping build is fatal (see
[`script_engine.cpp`](script_engine.cpp.md)). A rebuild that rejects nil here will terminate on
scripts that work in the original.

**Notes** — The type also teaches the binding layer how to *name* itself in a diagnostic —
`function<R>` — which is what makes an overload-resolution failure message readable and what the
bindings dump prints. That naming is part of the surface criterion 10 fixes only insofar as
tooling reads it; scripts do not.
