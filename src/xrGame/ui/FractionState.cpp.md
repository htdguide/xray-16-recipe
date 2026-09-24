# src/xrGame/ui/FractionState.cpp

> The earlier faction record, kept because one game's shipped scripts bind to its name and
> its fill function.

**Needs** — [`FractionState.h`](FractionState.h.md) · [Seam: Script virtual machine](../../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine) · [Seam: Script binding layer](../../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`FractionState.h`](FractionState.h.md)
**Tier floor** — T3: a record plus one script call

## Purpose

Same mechanism as [`FactionState.cpp`](FactionState.cpp.md) — the engine fills in identity and
standing, a named script function fills in everything else — for an earlier version of the
faction-war screen. It survives because the two are **distinct frozen script surfaces**, not
because the behaviour differs meaningfully.

A rebuild must ship both. Merging them and exporting one under two names is acceptable; making
one of the two script names disappear is not.

## State

```text
RECORD FractionState
  id             : text
  actor_goodwill : int
  name, icon, icon_big, target, target_desc, location : text
  member_count   : int
  resource       : real
  power          : real
  state_vs       : int        # script-filled; the war standing as a single number
  bonus          : int
```

The differences from [`FactionState`](FactionState.cpp.md), which are the only reason to read
this file:

| | `FractionState` | `FactionState` |
|---|---|---|
| war state | one integer, `state_vs` | five icon slots with five hints |
| script fill function | `pda.fill_fraction_state` | `pda.fill_faction_state` |
| exported class name | `FractionState` | `FactionState` |
| identifier property | `fraction_id` | `faction_id` |
| slot clearing before fill | none needed | required |

## `update_info`

**Contract** — Identical in shape to the sibling's, minus the clear: short-circuit on an empty
identifier, recompute the actor's goodwill toward this community from the relation registry
(0 when there is no current player entity), then hand the record to the named script function,
failing hard if that function is absent.

```text
FUNCTION update_info()
  IF id is empty THEN RETURN
  actor_goodwill = 0
  actor = current player entity
  IF actor exists
    actor_goodwill = relation_registry.community_goodwill(community_index_of(id), actor.id)

  fill = script_function("pda.fill_fraction_state")
  FAIL WITH "missing fraction fill function" IF fill is absent
  fill(this)
```

**Notes** — Because there are no slots to clear, a script that skips a field here leaves the
*previous* faction's value in place. That latent staleness is precisely what the successor
record's `ResetStates` was introduced to fix.

## `script_register`

**Contract** — Exports the record under the name `FractionState`. Frozen property names:
`fraction_id`, `actor_goodwill`, `name`, `icon`, `icon_big`, `target`, `target_desc`,
`location`, and the direct members `member_count`, `resource`, `power`, `state_vs`, `bonus`.
