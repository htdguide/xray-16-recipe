# src/editors/xrWeatherEditor/property_boolean_values_value.cpp

> A boolean shown as two words of the author's choosing, rather than as a checkbox.

**Needs** — [`property_boolean_values_value.hpp`](property_boolean_values_value.hpp.md) · [`property_boolean.cpp`](property_boolean.cpp.md) · [`property_converter_boolean_values.cpp`](property_converter_boolean_values.cpp.md)
**Used by** — [`property_boolean_values_value.hpp`](property_boolean_values_value.hpp.md)
**Tier floor** — T1: it converts two plain byte strings supplied by the unmanaged host into managed text at construction.

## Purpose

Some weather flags are not naturally yes/no — a sun flare's orientation, a sound channel's mode. Showing them as a checkbox forces the author to remember which way round it is. This form shows a two-item dropdown with labels the engine supplied.

It is [`property_boolean`](property_boolean.cpp.md) plus a label pair, and the pairing is the only content.

## State

```text
RECORD BooleanChoiceProperty EXTENDS BooleanProperty
  labels : list<text>       # exactly two; invariant: labels[0] means false, labels[1] means true
```

**Invariants** — exactly two labels, and **position is meaning**: the first is the false label, the second the true label. Nothing in the type enforces the count; the constructor reads exactly two and the setter asserts that a chosen label was found at position zero or one.

## Construction

**Contract** — copies the two supplied labels into managed text at construction time, once, rather than converting on every paint.

**Notes** — the conversion happens here because the labels arrive as plain byte strings from the unmanaged host and every conversion allocates (see [`pch.hpp`](pch.hpp.md)). Converting once at construction rather than per repaint is the decision; that a conversion is needed at all is incidental.

## `set`

**Contract** — takes the label the author picked, finds its position among the two, and writes the corresponding boolean.

```text
FUNCTION set(chosen : text)
  index = position of chosen in labels
  FAIL WITH "not one of the two labels" IF index is absent
  base.set(index == 1)
```

**Notes** — `get` is *not* overridden: it still returns the raw boolean, not a label. Turning that boolean into the right label on the way to the screen is [the converter's](property_converter_boolean_values.cpp.md) job, and it reads the label list off this object to do it.

That split — **the binding holds the data and the labels, the converter renders and the binding parses** — is consistent across every choice-valued property in this directory, and it is the one piece of structure a rebuild needs to notice, because it means the label list has two readers.
