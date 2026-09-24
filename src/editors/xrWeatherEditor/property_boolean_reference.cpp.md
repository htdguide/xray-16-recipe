# src/editors/xrWeatherEditor/property_boolean_reference.cpp

> The same yes/no cell, bound straight to the field instead of to a pair of callables.

**Needs** — [`property_boolean_reference.hpp`](property_boolean_reference.hpp.md) · [`property_holder_include.hpp`](property_holder_include.hpp.md) · [`xrSdkControls/Controls/Interfaces/IProperty.cs`](../xrSdkControls/Controls/Interfaces/IProperty.cs.md)
**Used by** — [`property_boolean_reference.hpp`](property_boolean_reference.hpp.md) · [`property_boolean_values_value_reference.cpp`](property_boolean_values_value_reference.cpp.md)
**Tier floor** — T1: it holds a reference to a field it does not own, for a lifetime it does not control.

## Purpose

The reference half of the binding matrix. Most weather fields need no computation on read or write — a fog density is just a number in a keyframe — so writing two trivial callables for each of them would be pure noise. This form binds the field.

## State

```text
RECORD BooleanFieldProperty
  target : reference to bool        # not owned
```

**Invariants** — **the referenced field must outlive the property.** Nothing enforces it. The editor upholds it structurally: a holder's properties describe one engine object, and the holder is destroyed before the object is. A rebuild that keeps this form must make the same guarantee, or must not offer the form.

**Notes** — the reference is stored in a small owned wrapper rather than inline, for the same reason [`property_boolean`](property_boolean.cpp.md) copies its callables: a managed object cannot hold something that needs releasing. The wrapper itself is a two-method shim — read the field, write the field — and carries no decision.

## `get` / `set`

**Contract** — read the field; narrow the edited value and write the field. No range, no validation, no notification: a write to the field *is* the change, and the running weather picks it up on the next frame because it reads the same field.

## Notes

The direct-reference form is faster and simpler, and it is also the form that cannot express anything: a field that must recompute a derived value on write, or mark its cycle dirty, needs the callable form. Which form a given weather field uses is decided at the point it is described, in [`property_holder_boolean.cpp`](property_holder_boolean.cpp.md).
