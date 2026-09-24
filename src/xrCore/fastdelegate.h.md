# src/xrCore/fastdelegate.h

> A callable reference that can hold a free function, a method bound to an object, or a lambda — the engine's event and callback currency, comparable and orderable so it can be stored in a set and removed again.

**Needs** — _(none of the engine's own)_
**Used by** — [`ide_impl.hpp`](../editors/xrWeatherEditor/ide_impl.hpp.md) · [`ppmd_compressor.h`](Compression/ppmd_compressor.h.md) · [`xr_sha.h`](Crypto/xr_sha.h.md) · [`xrCore.h`](xrCore.h.md) · [`xr_ini.h`](xr_ini.h.md)
**Tier floor** — T1: it stores a bound method as a fixed-size pair of an object pointer and a method pointer, with no allocation, which requires knowing the representation of a method pointer.

## Purpose

The engine passes callbacks everywhere: the event bus, the console's command handlers, the configuration parser's include filter, the UI's widget handlers, the scheduler's work items. It needs a callable value that is **cheap to copy, allocation-free, and comparable** — the last because a subscriber must be able to unsubscribe, and unsubscribing means finding the exact callback in a list.

A vendored third-party header. What it solves survives into any rebuild; how it solves it does not.

## State

```text
RECORD Delegate of (arguments...) -> result
  target : optional<address>    # the bound object, or nothing for a free function
  method : opaque bytes         # a method pointer, or a trampoline for a free function
```

**Invariants**

- **Fixed size, no allocation.** Constructing, copying and destroying a delegate is a couple of pointer moves. This is why it can be created per frame.
- **Two delegates are equal exactly when both halves match.** That is the property the event bus depends on: unsubscribing removes the one entry that equals the callback that was passed in.
- **A total order exists** over the two halves, so delegates can key an ordered container. It is address-based and therefore arbitrary but consistent within a run — the same caveat as every handle in this module.
- **The empty delegate is a distinguishable state** and calling it is undefined; the emptiness test exists so callers check first.

## Exported units

- **Construct** — empty; from a free function; from an object and one of its methods, const or not; from a lambda or other callable object.
- **Bind** — the same, after construction.
- **Invoke** — call with the declared arguments, returning the declared result.
- **Clear and test** — empty it, ask whether it is empty.
- **Compare** — equality, inequality, and a total order.
- **Opaque storage form** — a type-erased copy of the two halves, so that delegates of different signatures can be held in one collection and restored later. This is how the engine keeps heterogeneous handler tables.
- **Deduction helper** — build a delegate from an object and a method, or from a callable, without naming the signature.
- **Arity aliases** — names for delegates of zero through eight arguments, kept for older call sites; they are spellings of the general form.

## Notes

The header is a 2004 vintage and most of its bulk is compiler-specific machinery to make a bound method pointer fit in a fixed-size field on compilers that represent it in several ways. **All of that is incidental.** A rebuild on almost any tier gets this type from its standard library or its language, and the only decisions worth carrying are the three invariants above: no allocation, exact equality, and a total order.

Two of the arity aliases — those for seven and eight arguments — repeat one of their parameter types instead of listing the next, so they name the wrong signature. Nothing uses them; a rebuild that regenerates the aliases will not reproduce the defect.
