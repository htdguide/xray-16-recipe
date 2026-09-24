# src/editors/xrWeatherEditor/property_float.cpp

> One grid row bound to a real number in the engine by a getter and a setter, plus the step a nudge applies.

**Needs** — [`property_float.hpp`](property_float.hpp.md) · [`property_holder_include.hpp`](property_holder_include.hpp.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: owns native callback objects that must be released on a schedule the collector does not choose

## Purpose

The simplest and by far the most common property adapter: a row that reads one real from the engine when the grid asks and writes it back when the user commits. The base of the clamped and enumerated real adapters.

## State

```text
RECORD RealProperty
  getter : callback() -> real          # native; owned — a private copy, not an alias
  setter : callback(real)              # native; owned
  step   : real                        # multiplier applied to a nudge
                                       # invariant: > 0 wherever a nudge is offered
```

The callbacks are *copied* into storage this adapter owns. The engine hands them across as temporaries; keeping a reference to the caller's copy would outlive it.

## `construct(getter, setter, step)`

**Contract** — Takes private copies of the two callbacks. Allocates. The step is stored verbatim; the caller has already decided whether it is absolute or a fraction of a range ([`property_holder_float.cpp`](property_holder_float.cpp.md)).

## `release`

**Contract** — Frees the two callback copies. Runs exactly once, whether the grid releases the adapter explicitly when a row is dropped or the runtime reclaims it later.

**Notes** — This is the interop-boundary rule that every adapter in this directory repeats, and it is worth stating as a problem rather than a pattern. A managed object holding a foreign resource has two possible release triggers — the owner saying "done with this" and the collector noticing the object is unreachable — and they can arrive in either order or, for the first, not at all. The requirement is exactly: the foreign resource is freed once, and no later than process shutdown. Both triggers therefore route to the same body, and that body must tolerate running after the resource is already gone. A rebuild in a tier with deterministic destruction gets this for free and should not reproduce the two-path structure.

## `GetValue`

**Contract** — Calls the getter and hands the result to the grid. Does not clamp, cache or round — what the engine holds is what the row shows. Called on the user-interface thread while the engine may be rendering, so the getter must be cheap and must not block.

## `SetValue`

**Contract** — Converts the grid's value to a real and calls the setter. A value the grid supplies that is not a real is a programming error, not a user error — the grid has already parsed and type-checked against the declared type through the converter.

## `Increment(amount)`

**Contract** — Reads the current value, adds `amount × step`, writes it back. `amount` is the grid's own notion of one unit of nudge — one spin click, or one unit of mouse travel — so the step is what turns an interface gesture into a change in the authored quantity.

```text
FUNCTION Increment(amount)
  SetValue(GetValue() + amount * step)
```

**Notes** — Read-modify-write through the engine rather than accumulation in the adapter. That matters: the engine may clamp, quantise or reject the write, and the next nudge must start from what the engine actually holds, not from what the adapter last asked for. Drift between the row and the document is the failure this avoids.
