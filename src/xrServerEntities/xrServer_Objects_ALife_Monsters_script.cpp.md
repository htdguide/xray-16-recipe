# src/xrServerEntities/xrServer_Objects_ALife_Monsters_script.cpp

> Exports the identity mixin — faction, profile, name, rank, reputation, portrait — plus the trader, the two zone levels and the rat.

**Needs** — [`xrServer_Objects_ALife_Monsters.h`](xrServer_Objects_ALife_Monsters.h.md) · [`specific_character.h`](specific_character.h.md) · [`xrServer_script_macroses.h`](xrServer_script_macroses.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2.

## Purpose

The first of four parts of the creature export. One registration here carries real surface —
the identity mixin, which is what every dialogue, relation and trading script in the game
reads — and the rest are declarations.

## `cse_alife_trader_abstract`

**Contract** — the identity mixin. Six read accessors and three writes.

- **`community`** — the faction's *name*, resolved from the stored index. The index is never
  exported; a script always deals in names.
- **`profile_name` / `set_profile_name`** — the authored template this character was drawn
  from. Writable, which is how a script re-templates a character.
- **`character_name` / `set_character_name`** — the display name. Writable, and the write
  takes plain text.
- **`rank` / `set_rank`** and **`reputation`** — reputation is read-only; rank is not.
- **`character_icon`** — the portrait.

**Invariants** — **`rank`, `set_rank` and `reputation` each force profile resolution first.**
Reading a rank on a record whose individual has not been chosen chooses one, with all the
side effects that entails (see
[`xrServer_Objects_ALife_Monsters.cpp`](xrServer_Objects_ALife_Monsters.cpp.md)): a visual,
a faction, a name and a wallet are assigned. A script reading a rank can therefore change
what a character looks like. That is not a bug — it is lazy resolution — but it is invisible
from the script side and a rebuild should consider resolving eagerly at spawn instead.

**`character_icon` resolves *and then repairs*.** It forces resolution, and if the portrait
is still empty it loads the individual's authored data and copies the portrait across. That
second step exists because the portrait is filled only on the path through adoption, and a
record that arrived from a save with an individual already set never took that path. It is a
lazy backfill for a field the save read leaves empty.

**Notes** — four of the accessors go through small adapters that convert interned strings to
plain text. No behavioural difference. The constructor is present and commented out: the
mixin is never built alone.

## `cse_alife_trader`

**Contract** — the shopkeeper record: a dynamic visual object and an identity. No added
surface.

## `cse_custom_zone` / `cse_anomalous_zone`

**Contract** — the two zone levels, at the dynamic-alife level with no added surface.

**Notes** — the script names drop the `alife` segment that every neighbouring record carries
(`cse_custom_zone`, not `cse_alife_custom_zone`). A naming inconsistency frozen by
conformance criterion 10.

## `cse_alife_monster_rat`

**Contract** — the rat, at the **monster** level, declared with both its bases: a monster and
an inventory item, because its corpse can be carried.
