# src/xrServerEntities/script_value_wrapper.h

> Binds one shadow cell to one typed field of one script table, with a hand-written case for the two types that do not survive a straight copy.

**Needs** — [`script_value.h`](script_value.h.md) · [`script_value_wrapper_inline.h`](script_value_wrapper_inline.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`script_properties_list_helper.cpp`](script_properties_list_helper.cpp.md) · [`script_value_wrapper_inline.h`](script_value_wrapper_inline.h.md)
**Tier floor** — T2: a typed cell whose stored representation is chosen per type, and whose address is handed to a serializer.

## Purpose

[`script_value.h`](script_value.h.md) says what a shadow cell must do; this says what it
holds. For most types the answer is the obvious one — store a value of that type, read it
from the table at construction, push it back on write-back, and expose its address. Two
types need more than that, and the two exceptions are the only load-bearing content in the
file.

## State

```text
RECORD ScriptValueCell OF T
  INHERITS ScriptValue
  value : T      # the shadow copy; its address is what the editor and the serializers use
```

## the generic cell

**Contract** — construction reads the named field from the table and converts it to the
cell's type; a conversion failure is the binding layer's error, raised there. Write-back
assigns the stored value to the table field. The cell exposes its value's **address**, not
its value — that is the whole purpose, and it means the cell must not be moved or copied
once a property handle or a serializer field reference points at it.

## the boolean cell

**Contract** — reads a script boolean, but **stores it in an integer-width field**, and
converts back to a true boolean on write-back.

**Notes** — the width is not arbitrary. The editor's property model and the record
serializers both work in terms of a machine word for flags and booleans, and handing them
the address of a one-byte value would have them read three bytes of neighbouring storage.
The conversion on write-back is what stops a nonzero-but-not-one integer reaching script as
something other than `true`. A rebuild whose boolean is already word-sized, or whose
property model takes a boolean directly, does not need this case at all.

## the string cell

**Contract** — reads a script string and stores it as an interned string. Write-back
**substitutes the empty string for an unset cell** before assigning.

**Notes** — the substitution is the interesting line. An interned string that was never
assigned is the absent value, and pushing *that* into a script table yields nothing rather
than an empty string, which is a different thing to script code: a field that vanishes fails
a `#` length test and an equality test differently from one holding `""`. Coercing at the
boundary means a script record's string field always exists after a write-back, whatever the
engine did to it in between. This is a real behavioural rule, not a null check.

## Notes

The two-level split — a cell type and a thin type derived from it — exists so that the
specialized cases can replace the base while the derived name stays the same for every
caller. It is a C++ mechanism for "override the storage but keep the name" and carries no
decision of its own.
