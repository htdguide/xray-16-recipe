# src/editors/xrWeatherEditor/property_float_enum_value.cpp

> A real that may only take one of an authored set of magnitudes, chosen by name.

**Needs** — [`property_float_enum_value.hpp`](property_float_enum_value.hpp.md) · [`property_float.hpp`](property_float.hpp.md)
**Used by** — reached through its declarations in [`property_float_enum_value.hpp`](property_float_enum_value.hpp.md); callers name that, not this file.
**Tier floor** — T2: copies a native `(magnitude, label)` array into a managed list at construction

## Purpose

Where a real quantity in the weather document is really a mode — a handful of magnitudes that mean something, with names — this adapter presents the names and stores the magnitude. The list is fixed when the property is registered.

## State

```text
RECORD RealChoiceProperty EXTENDS RealProperty
  choices : list<Pair<real, text>>     # invariant: non-empty; entry 0 is the fallback
                                       # invariant: magnitudes are distinct
```

The native array of pairs is copied into a managed list at construction — labels included — so nothing in this adapter depends on the engine's array outliving the registration call.

## `construct(getter, setter, choices, count)`

**Contract** — Copies `count` `(magnitude, label)` pairs into the list, converting each label to managed text. The step inherited from the base is set to the default `0.05` and then never used, because nudging is suppressed. Allocates.

## `GetValue`

**Contract** — Reads the bound real. If it matches a listed magnitude exactly, returns it; otherwise returns the *first* listed magnitude.

```text
FUNCTION GetValue() -> real
  current = base.GetValue()
  FOR EACH choice IN choices
    IF choice.first == current THEN RETURN current
  RETURN choices[0].first          # the document holds an unlisted magnitude
```

**Notes** — The comparison is exact equality on a real, which is the load-bearing fragility here: a magnitude that survived a round trip through the document's text form may no longer compare equal to the one in the list, and the row then silently shows the first choice instead of what the document holds. This is tolerable only because the listed magnitudes are authored round numbers written and read back by the same formatter. A rebuild should compare against the *written* form, or index the choice rather than store its magnitude.

The fallback to entry zero — rather than reporting the mismatch — is why the list must never be empty. Registration with an empty list has no defined behaviour anywhere in this layer.

## `SetValue`

**Contract** — The grid supplies the *label* the user picked. Finds the matching label and writes its magnitude; falls back to the first choice's magnitude when the label matches nothing.

```text
FUNCTION SetValue(label)
  FOR EACH choice IN choices
    IF choice.second == label THEN
      base.SetValue(choice.first)
      RETURN
  base.SetValue(choices[0].first)
```

**Notes** — Read yields a magnitude and write takes a label: the two directions of this adapter are not symmetric, and that asymmetry is deliberate. It is the converter that turns the magnitude the row reports into a label to display and hands the picked label back raw ([`property_converter_float_enum.cpp`](property_converter_float_enum.cpp.md)), so the adapter holds the list and the converter holds the presentation. A rebuild that keeps them in one place must still perform both mappings.

## `Increment`

**Contract** — Does nothing. A choice has no ordering the user authored, so a nudge has no meaning; suppressing it rather than omitting the increment protocol keeps the grid from having to special-case the row.
