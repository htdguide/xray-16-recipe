# src/xrGame/mixed_delegate.h

> One callback slot that a script or the engine may fill interchangeably — the mechanism by which an asynchronous operation reports back to whichever side asked for it.

**Needs** — [`mixed_delegate_unique_tags.h`](mixed_delegate_unique_tags.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer) · [Seam: Multiplayer matchmaking and accounts](../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)
**Used by** — [`account_manager.cpp`](account_manager.cpp.md) · [`account_manager.h`](account_manager.h.md) · [`login_manager.h`](login_manager.h.md) · [`login_manager_script.cpp`](login_manager_script.cpp.md) · [`profile_data_types_script.h`](profile_data_types_script.h.md)
**Tier floor** — T3: a two-sided callback slot; no implementation file

## Purpose

The account and matchmaking operations are asynchronous: log in, look up a profile, suggest
a nickname, buy something in the store. Each is started with a completion callback, and the
caller is sometimes the engine's own multiplayer screens and sometimes a script. This type is
the slot that accepts either.

It solves exactly one problem, and the problem is the interesting part: **the callee must not
know which side it is calling back into**. Without this, every asynchronous operation would
need two entry points and two result paths, and a script could not start an operation the
engine also starts.

The slot carries **two** possible bindings — a direct call into an engine object, and a
script function together with the script object it is a method of — and invoking the slot
tries them in that order. Exactly one is expected to be filled; both empty is a hard failure
at the call site rather than a silent no-op, because an asynchronous operation with nowhere
to report to has already lost its result.

## State

```text
RECORD MixedDelegate<signature>
  engine_target : optional<bound method>     # an object and one of its methods
  script_target : optional<bound function>   # a script object and a function on it
```

**Invariants**
- The engine binding wins when both are present. Nothing sets both, so the precedence is a
  tiebreak that never fires; a rebuild may also treat both-set as an error.
- Emptiness is tested by asking each binding whether it holds anything; the slot is truthy
  when either is.
- Clearing clears both.

## The signature

The slot as written admits **exactly two parameters** and any return type. That is not a
principled arity — it is the arity every account callback happens to have (a result code and
a payload). A rebuild with variadic generics writes this once for any arity, which is what
the source itself notes it should have done.

## `bind` / `clear` / invocation / the truth test

**Contract** — `bind` has two forms, one per side, and replaces whatever that side held.
`clear` empties both. Invocation dispatches to the engine binding if present, else the script
binding, and **fails the process** if neither is. The truth test reports whether either side
is filled.

```text
FUNCTION invoke(a, b) -> R
  IF engine_target EXISTS THEN RETURN engine_target(a, b)
  IF script_target EXISTS THEN RETURN script_target(a, b)
  FAIL WITH "mixed delegate is not bound"
```

**Notes** — the failure is deliberate and is the right call for this mechanism: an unbound
completion callback means an operation was started by code that forgot to say where the
answer goes, which is a programming error that would otherwise present as an account
operation that silently never completes.

## `script_register`

**Contract** — each concrete slot type exposes itself to scripts under a name: a default
constructor, a constructor taking a script object and function, a `bind` for the same pair,
and `clear`. The engine-side binding is deliberately **not** exported — a script may not
point a callback at an engine method. The registration is generated identically for every
slot type from one definition, which is the whole of what the macro does.

## The unique tag

Each slot type carries a numeric tag in addition to its signature, defaulting to "none".
Several account callbacks share the same signature — a result code and a string — and would
therefore be the *same* type, so the script layer could not give them distinct names and the
binding registration would collide. The tag exists purely to make them distinct types. The
tag values are in
[`mixed_delegate_unique_tags.h`](mixed_delegate_unique_tags.h.md).

This is an artifact of a language where type identity drives name binding. A rebuild whose
script binding registers by explicit name rather than by type deletes the tag entirely — and
should, because the tag is otherwise meaningless data attached to a callback slot.
