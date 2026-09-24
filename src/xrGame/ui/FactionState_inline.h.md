# src/xrGame/ui/FactionState_inline.h

> The field accessors of the faction record, including the five war-state slots spelled out
> one accessor pair per slot.

**Needs** — [`FactionState.h`](FactionState.h.md)
**Used by** — [`FactionState.h`](FactionState.h.md)
**Tier floor** — T3: field accessors

## Purpose

Pure accessors over the record declared in [`FactionState.h`](FactionState.h.md): each is a
read or a write of one field, and none of them decides anything. The file exists because the
record is exported to the script layer and the binding needs a callable pair per property.

## The one decision in the file

The five war-state slots are reachable two ways: by index, and through five separately named
accessor pairs — `war_state1` through `war_state5`, and the same for the hints. The indexed
form is what the screen uses; the named form exists **only** because the script binding layer
cannot express an indexed property, and the script that fills the record must assign each slot
by name.

That is the whole reason this file is a hundred lines rather than ten, and it is load-bearing
in one direction: the five names are part of the frozen script surface (conformance criterion
10) and cannot be collapsed into an array without breaking every shipped faction script. A
rebuild whose binding layer *can* express an indexed property still owes the five names.

Indexed access asserts the index is in range; the named accessors cannot be out of range by
construction, which is the other half of why both forms exist.
