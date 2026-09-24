# src/xrServerEntities/xrServer_Objects_ALife_script.cpp

> Exports the alife record levels: the scheduling mixin, the graph point, the alife object with its online/offline controls, the dynamic levels, the restrictor, the level changer and the container.

**Needs** — [`xrServer_Objects_ALife.h`](xrServer_Objects_ALife.h.md) · [`xrServer_script_macroses.h`](xrServer_script_macroses.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2.

## Purpose

The middle of the record export: the levels between "a record" and "a concrete thing". This
is where the surface a mod actually uses lives, because the online/offline controls are here
and moving entities between online and offline is most of what an alife mod does.

## `cse_alife_object`

**Contract** — the alife record. Exported at the alife level, so a script subclass may
override the five simulation predicates as well as the serializations. Plus:

- **`online`** — read-only: is this record currently simulated in detail.
- **`move_offline`** — in two forms, a read and a write. The write is the request that sends
  an entity offline.
- **`visible_for_map`** — read and write: whether the entity shows on the map.
- **`can_switch_online` / `can_switch_offline` / `use_ai_locations`** — **write only**. The
  matching reads exist natively but are exported only through the overridable predicate set,
  so a script sets the flag through one name and reads it through another.
- **`m_level_vertex_id` / `m_game_vertex_id`** — read-only: where the entity is, on the
  navigation graph and the game graph. These are the two numbers an offline position is made
  of.
- **`m_story_id`** — read-only: the authored story identifier, which is how a quest script
  finds a specific entity across a whole game.

**Invariants** — the paired read/write forms are exported under **one name distinguished by
arity**: called with no argument it reads, called with a boolean it writes. That is a
convention the binding layer supports and shipped scripts rely on; a rebuild whose binding
layer resolves by arity can reproduce it, and one that cannot must pick two names and break
compatibility.

**The vertex identities are read-only.** A script may not teleport an entity by writing its
graph vertex — that goes through the simulation, which must update the registries that index
entities by location.

## `cse_alife_schedulable` / `ipure_schedulable_object`

**Contract** — the scheduling mixin and the interface below it, both opaque. A script can
tell a record is scheduled; the scheduling parameters are not exported.

## `cse_alife_graph_point`

**Contract** — the game-graph point record, exported at the abstract level. It is authored
data that defines the game graph's vertices and is not itself simulated, which is why it sits
at the abstract level rather than the alife one.

## `cse_alife_group_abstract`

**Contract** — the group mixin, opaque.

## `cse_alife_dynamic_object` / `cse_alife_dynamic_object_visual`

**Contract** — the two levels that add "is simulated" and "has a model", both exported at
the dynamic-alife level so a script subclass gets the life-cycle notifications.

## `cse_alife_ph_skeleton_object`

**Contract** — a dynamic visual object with a ragdoll.

## `cse_alife_space_restrictor`

**Contract** — a dynamic object with a shape: a volume that constrains where entities may go.
See [`restriction_space.h`](restriction_space.h.md).

## `cse_alife_level_changer`

**Contract** — a restrictor that moves the player to another level, plus one read-only
property: the destination level's name.

**Notes** — the property is exported through a small adapter because the stored value is an
interned string and the script side wants plain text. Only the name is exposed; the
destination position and orientation, which the record also holds, are not — a script can
tell *where a door leads* but not *where it puts you*.

## `cse_alife_inventory_box`

**Contract** — a container: a dynamic visual object holding other records. Exported with no
additional surface; its contents are reached through the generic parent/child relationship on
the abstract record.
