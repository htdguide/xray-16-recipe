# src/xrGame/ui/FactionState.cpp

> One faction's row on the faction-war screen: the engine supplies identity and standing, a
> script supplies everything else.

**Needs** — [`FactionState.h`](FactionState.h.md) · [Seam: Script virtual machine](../../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine) · [Seam: Script binding layer](../../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`FactionState.h`](FactionState.h.md)
**Tier floor** — T3: a record plus one script call

## Purpose

The faction-war screen shows, per faction, a mixture of things the engine knows (who you are
on good terms with) and things only the game's own scripts know (how many members a faction
has, what it currently wants, how the war is going). Rather than teach the engine the war
rules, this record is passed *into* a named script function that fills it in. The engine's
contribution is the identity and the goodwill; everything else is the script's.

That inversion is the file's whole point, and a rebuild must preserve it: the war rules live
in shipped Lua, not in the engine.

## State

```text
RECORD FactionState
  id             : text        # the community identifier, e.g. "stalker"
  actor_goodwill : int         # engine-computed, see update_info

  name           : text        # display name
  icon           : text        # small icon
  icon_big       : text        # large icon
  target         : text        # current objective
  target_desc    : text        # its description
  location       : text

  member_count   : int         # \
  resource       : real        #  | script-filled; the engine never computes these
  power          : real        #  |
  bonus          : int         # /

  war_state      : text [5]    # icon name per slot
  war_state_hint : text [5]    # hint per slot
```

Invariants:

- An empty `id` means "no faction selected"; every refresh short-circuits on it, and the
  record is then meaningless rather than empty-but-valid.
- The ten war-state slots are **cleared before** the script runs, so a script that fills three
  slots leaves two genuinely empty rather than showing the previous faction's icons. This is
  the only reason `ResetStates` exists as a separate operation.

## `update_info`

**Contract** — Refresh the record. Returns nothing; mutates in place. Fails hard if the named
script function is absent, because a faction screen with no script behind it would show blanks
with no explanation.

```text
FUNCTION update_info()
  IF id is empty THEN RETURN

  actor_goodwill = 0
  actor = current player entity
  IF actor exists
    community = community_index_of(id)
    actor_goodwill = relation_registry.community_goodwill(community, actor.id)

  clear all war_state and war_state_hint slots

  fill = script_function("pda.fill_faction_state")
  FAIL WITH "missing faction fill function" IF fill is absent
  fill(this)                     # the script writes the remaining fields
```

**Invariants** — The order is fixed: goodwill first (so the script can read it), slots cleared
second, script last (so the script's writes survive). Reordering the clear after the call
erases the script's work — which is exactly the bug the separation invites.

**Notes** — Goodwill is read from the relation registry rather than stored: it changes as the
player acts, and the screen must show the live value. When there is no current player entity —
the screen can be opened in states where there is not — goodwill reads as 0, meaning neutral,
rather than being left stale.

## `script_register`

**Contract** — Exports the record to the script layer under the name `FactionState`, with
the four numeric fields as direct read/write members and everything else as named properties.
The property names are part of the frozen script surface: `faction_id`, `actor_goodwill`,
`name`, `icon`, `icon_big`, `target`, `target_desc`, `location`, `war_state1`…`war_state5`,
`war_state_hint1`…`war_state_hint5`. A rebuild must reproduce these names exactly or the
shipped faction scripts stop filling the screen.

## `ResetStates`

**Contract** — Clears the ten war-state slots. See the invariant above.
