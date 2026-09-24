# src/xrServerEntities — the authoritative entity records

Chapter 22 of [`SYSTEM-REQUIREMENTS.md`](../../SYSTEM-REQUIREMENTS.md#7-build-order).

This directory defines what an entity **is**: on disk, in a save, and on the wire. Every
spawn record in every shipped level is one of these types serialized; every save game is a
list of them; every network spawn and update message is one of them again in a third
encoding. Two of the conformance criteria rest directly here — a save from the original
engine must load (criterion 7) and the entire exported script surface must be unchanged
(criterion 10) — and the frozen spawn format is this chapter's output.

The word for the thing defined here is **server object**: the authoritative record of an
entity — its identity, its position, its class-specific state, everything that must survive
a save or reach a network client. It exists whether or not the entity is currently
simulated. Its counterpart, the **client object**, is the live, renderable, physically
simulated instance, and it lives in chapter 23. In single player both exist in one process,
which is the single most confusing thing about this codebase for a newcomer; the names come
from the multiplayer architecture and are used everywhere regardless.

Nothing in this directory renders, simulates or thinks — with one deliberate exception. A
creature's **offline brain** lives here, because the coarse alife simulation advances a
creature *through its record* while it is offline. That is the design, not an accident: an
entity nobody is looking at is nothing but its record plus forty lines of decision.

## Where it sits

It rests on the core layer (virtual filesystem, configuration parser, interned strings),
the maths layer, the network layer's bit-packed buffer, the navigation graphs from chapter
14, and the script engine and binding layer. It is built before the game.

It forms a **cycle with chapter 23**, and the cycle is real: the class registry in this
directory names every gameplay class in that one, because the registry's whole job is to
pair a record type with a live object type. The build resolves it by compiling this
directory's sources *into* the game module rather than as a library of its own. In a rebuild
this is one shared data module with two consumers and a registration table that both
contribute to — not a cycle.

## Compiled twice, with different macro sets

The same sources are compiled into the game and into the offline tools (the spawn-file
compiler and the level editor). Three macro switches decide what a build gets, and the
differences are worth stating precisely because "it's just the editor" undersells them.

| Switch | Build | What changes |
|---|---|---|
| tools build | the spawn compiler | The class registry is absent: a record built here does **not** resolve its script class number, and the object-factory lookup is compiled out. So is the character's specific-character resolution. The records themselves, and their read and write, are identical. |
| editor build | the level editor | Adds read and write of every record **against a configuration section** as well as against a binary stream — a fourth serialization, used for authoring — and the legacy spawn-format read path. |
| shipping build | the retail game | Removes every record's property-list contribution, the debug spawner, and the whole editor property model. |

The invariant across all three is that **the binary record layout is identical**. Only the
surfaces around it move. A rebuild that does not need an editor can drop the property model
and the configuration-backed serialization entirely — but must keep in mind that the
configuration-backed path silently **omits some fields** that the binary path writes (see
[`alife_human_brain.cpp`](alife_human_brain.cpp.md)), so a record round-tripped through
configuration is not byte-equal to one round-tripped through a packet.

## Load-bearing ideas, named once

### The class identifier is eight characters reinterpreted as a number

An entity's type is named by a 64-bit value built from eight ASCII characters, most
significant first, right-padded with spaces. `"AI_STL  "` is a stalker. The values are in
[`clsid_game.h`](clsid_game.h.md), they appear verbatim in every shipped level's spawn
file, and they are therefore an **input** to the rebuild, not a decision it makes.

There is a *second* class identity: the record's position in the sorted registry, a small
integer, which is what scripts compare against. It never reaches disk or the wire, and it is
allowed to be position-derived for exactly that reason. Confusing the two is the most
likely serious mistake in this chapter. See
[`object_factory_inline.h`](object_factory_inline.h.md).

### An entity is (class, section)

The class supplies the behaviour and the serialization; the **section** — a named block in
the `ltx` configuration — supplies every number. A record cannot be constructed without a
section, because it reads its defaults out of one during construction, including its own
class identifier (from the section's `class` key). One class serves dozens of sections:
every magazine-fed weapon in the game shares one record type and differs only in its
section.

### The entity identifier is sixteen bits, and all-bits-set means "none"

An entity is named across the save file, the network protocol and every internal registry by
a 16-bit handle. It caps the world at 65 535 entities. `0xffff` is the in-band "no entity"
value: an entity with no parent stores `0xffff` in its parent field, and the same value
means "not registered with a smart terrain", "no phantom" and so on. Widening it breaks the
save format and the protocol together. See [`alife_space.h`](alife_space.h.md) for this and
every other width in the chapter.

### One record, three serializations — and they are genuinely different

This is the subtle part of the chapter. Every record type implements three pairs of
read/write, and a rebuild that collapses them will not load a shipped save.

| Pair | When | What it carries |
|---|---|---|
| **spawn** | reading a level's spawn file; sending a newly created entity to a client | The *authored* state: section name, display name, position, orientation, the entity identifier and its parent and phantom, the respawn time, the state flags, the game-mode mask, the spawn-file format version, the script class version, and an opaque blob of client-side custom data. This is the record as the level designer wrote it. |
| **state** | writing and reading a save game | The *full current* state: everything the spawn record has, plus every field the entity has acquired since — inventory contents, health, the offline brain's preferences, physics snapshots, script-declared values. Read takes an explicit byte count, so a record can be skipped without being understood. |
| **update** | the per-tick network message for an entity already spawned | The *volatile* state only, quantized: position, orientation, velocity, a small set of per-class values. It is lossy by design and carries nothing that could be re-derived. |

Three consequences a rebuild must respect:

1. **The spawn record's tail is an opaque blob.** After the generic header the class writes
   its own payload, and the container records the payload's length. A reader that does not
   recognize a class can still skip its record. That is what lets the tools build, which
   knows fewer classes than the game, read a complete spawn file.
2. **Save reading is version-gated field by field.** Every record carries the spawn-format
   version it was written under, and each class's read walks a ladder of "if the version is
   at least *n*, this field is present". There is no per-field framing, so an obsolete field
   must be *parsed and discarded*, never skipped by byte count. The current version is
   **128**, and the changelog of what each revision added is kept as a comment block at the
   top of [`xrServer_Objects.h`](xrServer_Objects.h.md) — which is the single most valuable
   page in the chapter for anyone implementing the read side.
3. **Update and state disagree about precision on purpose.** A position is three full floats
   in a save and a quantized triple on the wire. The bounds used for quantization are a
   convention between writer and reader that the stream does not carry; getting them wrong
   produces silently wrong values rather than an error. See
   [`PHNetState.cpp`](PHNetState.cpp.md), which is the clearest worked example.

The save format is versioned as a whole too, at 7, and a mismatched save is **refused
rather than guessed at** — the right call for a format with no framing.

### The record hierarchy, and what each level adds

The inheritance chain is deliberately much shallower than the behaviour hierarchy in chapter
23, because every extra record type is another on-disk shape to keep frozen forever.

```text
abstract record                  identity, placement, class, section, spawn flags,
                                 game-mode mask, the script value container
  └─ temporary                   + a navigation vertex; exists only while online
                                   (rockets, grenades in flight)
  └─ alife object                + a story identifier, a spawn identifier, the online flag
     └─ dynamic object           + a game-graph vertex, a level vertex, a distance
                                   travelled — the things that let it exist offline
        └─ dynamic visual        + a model reference
        └─ creature              + health, team, squad, group, the death time
           └─ monster            + the offline brain, the smart-terrain identifier
              └─ human           + the human brain, the character profile, rank,
                                   reputation, community, money, the known-info list
        └─ inventory item        + condition, slot placement, the upgrade list
           └─ weapon             + ammunition state, attached addons, the fire mode
        └─ space restrictor      + a shape set and a restrictor kind
           └─ zone               + the anomaly's own state
              └─ smart terrain   + the job table
```

Two mixins are inherited alongside rather than in the chain: a **shape** (a set of spheres
and boxes, [`ShapeData.h`](ShapeData.h.md)) and a **physics skeleton** (a ragdoll snapshot,
[`PHNetState.h`](PHNetState.h.md)). Both carry their own serialization and both are
composed into several branches of the tree.

### The script-visible surface is frozen

Scripts read and write these records directly, and conformance criterion 10 freezes the
whole exported surface — class names, method names, property names, and the order callbacks
fire. Four things are exported from this directory:

- **the records themselves**, with their fields as readable and writable properties;
- **the class-number enumeration**, published as `clsid.<name>` from the registry's sorted
  order;
- **the serialization primitives** — the wire buffer with every width and quantization it
  supports ([`script_net_packet_script.cpp`](script_net_packet_script.cpp.md)), the
  configuration file, the flag words, the vector, the transform and the colour — so that a
  script-declared record can serialize itself into exactly the same stream;
- **the ability to declare a new class entirely in script**, registered against a new eight-
  character tag, with Lua constructors for both halves
  ([`object_item_script.cpp`](object_item_script.cpp.md)).

A script-declared record stores its state in a Lua table. The **value container** on every
record is what bridges that: shadow cells that hold a copy of a table field, owned by the
record, written back on serialization, and addressable by the editor. See
[`script_value_container.h`](script_value_container.h.md) and
[`script_properties_list_helper.cpp`](script_properties_list_helper.cpp.md).

### Records are edited by hand

Every record contributes rows to a property list so the level editor can edit its fields
in place, with multi-selection, per-type validation and revert. That surface is the reason
[`PropertiesListTypes.h`](PropertiesListTypes.h.md) and
[`xrEProps.h`](xrEProps.h.md) are in this directory rather than in an editor module: the
records are compiled into both, so the property vocabulary has to be shared. It is compiled
out of the shipping build entirely.

## The files

| File | Role |
|---|---|
| [`xrServer_Objects_Abstract.h`](xrServer_Objects_Abstract.h.md) · [`.cpp`](xrServer_Objects_Abstract.cpp.md) | The interfaces a record satisfies: entity, shape, visual, motion, and the editor-facing owner |
| [`xrServer_Object_Base.h`](xrServer_Object_Base.h.md) · [`.cpp`](xrServer_Object_Base.cpp.md) | The abstract record: identity, placement, class, section, spawn read/write, the cast-to-facet table |
| [`xrServer_Objects.h`](xrServer_Objects.h.md) · [`.cpp`](xrServer_Objects.cpp.md) | The spawn-format version changelog, the shape mixin, the physics-skeleton mixin, the spectator and temporary records |
| [`xrServer_Objects_ALife.h`](xrServer_Objects_ALife.h.md) · [`.cpp`](xrServer_Objects_ALife.cpp.md) | The alife record levels: object, dynamic object, dynamic visual, graph point, restrictor, zone, smart terrain |
| [`xrServer_Objects_ALife_Items.h`](xrServer_Objects_ALife_Items.h.md) · [`.cpp`](xrServer_Objects_ALife_Items.cpp.md) | Inventory-item records: condition, placement, upgrades; weapons, ammunition, grenades, artefacts, outfits, detectors, the PDA |
| [`xrServer_Objects_ALife_Monsters.h`](xrServer_Objects_ALife_Monsters.h.md) · [`.cpp`](xrServer_Objects_ALife_Monsters.cpp.md) | Creature records: health and team, the monster and human levels, the trader, the group, the online/offline group |
| [`xrServer_Objects_Alife_Smartcovers.h`](xrServer_Objects_Alife_Smartcovers.h.md) · [`.cpp`](xrServer_Objects_Alife_Smartcovers.cpp.md) | The smart-cover record: an authored cover position with its loopholes and transitions |
| [`xrServer_Objects_ALife_All.h`](xrServer_Objects_ALife_All.h.md) | One include that pulls in every record type; the registration list's entry point |
| [`xrServer_Factory.cpp`](xrServer_Factory.cpp.md) | Creating a record from a section name alone, by reading the section's class key |
| [`xrServer_Space.h`](xrServer_Space.h.md) | The build switch that adds or removes every record's editor methods, plus record destruction |
| [`xrMessages.h`](xrMessages.h.md) | The network message identifiers these records are carried in |
| [`clsid_game.h`](clsid_game.h.md) | **The frozen class-identifier registry**: every eight-character tag the shipped data uses |
| [`object_factory.h`](object_factory.h.md) · [`.cpp`](object_factory.cpp.md) | The registry's surface and lifecycle |
| [`object_factory_inline.h`](object_factory_inline.h.md) | The registry's algorithms: lazy creation, sorted insertion, binary lookup, the script class number |
| [`object_factory_impl.h`](object_factory_impl.h.md) | Choosing an entry shape from a class's ancestry at registration time |
| [`object_factory_register.cpp`](object_factory_register.cpp.md) | **The registration list**: every tag paired with its record type and its live object |
| [`object_factory_script.cpp`](object_factory_script.cpp.md) | Publishing the registry to scripts; registering script-declared classes |
| [`object_factory_space.h`](object_factory_space.h.md) | Names the two base types the registry deals in |
| [`object_item_abstract.h`](object_item_abstract.h.md) · [`_inline.h`](object_item_abstract_inline.h.md) | What a registry entry is: tag, script name, two constructors |
| [`object_item_client_server.h`](object_item_client_server.h.md) · [`_inline.h`](object_item_client_server_inline.h.md) | The paired entry, and the one that switches between single-player and multiplayer classes |
| [`object_item_single.h`](object_item_single.h.md) · [`_inline.h`](object_item_single_inline.h.md) | The entry for a class that has only a record or only a live object |
| [`object_item_script.h`](object_item_script.h.md) · [`.cpp`](object_item_script.cpp.md) | The entry whose constructors are script functions, and its error containment |
| [`object_factory_spawner.h`](object_factory_spawner.h.md) · [`.cpp`](object_factory_spawner.cpp.md) | Classifying every configuration section into a spawnable category; the developer spawn panel |
| [`alife_space.h`](alife_space.h.md) · [`.cpp`](alife_space.cpp.md) | **Every identifier width, the save-file chunk numbers, and the frozen enumerations**; hit-type name conversion |
| [`alife_monster_brain.h`](alife_monster_brain.h.md) · [`.cpp`](alife_monster_brain.cpp.md) · [`_inline.h`](alife_monster_brain_inline.h.md) | The offline decision cycle: choose a smart terrain, take its job, walk the game graph |
| [`alife_human_brain.h`](alife_human_brain.h.md) · [`.cpp`](alife_human_brain.cpp.md) · [`_inline.h`](alife_human_brain_inline.h.md) | The human addition: equipment tastes, money, and the version-gated read of all three games' saves |
| [`alife_movement_manager_holder.h`](alife_movement_manager_holder.h.md) | What "an offline entity's position" is: two graph vertices and the progress between them |
| [`character_info.h`](character_info.h.md) · [`.cpp`](character_info.cpp.md) | Resolving "profile *X*" into a named individual, filling only what the record left unset |
| [`character_info_defs.h`](character_info_defs.h.md) | Goodwill, class, reputation, rank, community — their ranges, and "unset" versus "neutral" |
| [`specific_character.h`](specific_character.h.md) · [`.cpp`](specific_character.cpp.md) | The authored individual: name, portrait, biography, faction, dialogue set, starting inventory |
| [`InfoPortionDefs.h`](InfoPortionDefs.h.md) | One piece of knowledge a character has, and when they learned it |
| [`xml_str_id_loader.h`](xml_str_id_loader.h.md) | The identifier-to-file-position index that makes "find profile *X*" cheap across thousands of XML entries |
| [`shared_data.h`](shared_data.h.md) | Sharing one loaded template across every instance that names it, with reference counting |
| [`PHNetState.h`](PHNetState.h.md) · [`.cpp`](PHNetState.cpp.md) | The rigid-body snapshot in three precisions, and the ragdoll's worth of them |
| [`PHSynchronize.h`](PHSynchronize.h.md) | What a physically simulated object must offer so its state can enter and leave a record |
| [`ShapeData.h`](ShapeData.h.md) | The volume an entity occupies when it is not a model: spheres and transformed boxes |
| [`gametype_chooser.h`](gametype_chooser.h.md) · [`.cpp`](gametype_chooser.cpp.md) | Which game modes an authored entity exists in, and reading that from both format generations |
| [`game_base_space.h`](game_base_space.h.md) | Player flags and match phases, shared between the rules and the records |
| [`inventory_space.h`](inventory_space.h.md) | The numbered equipment slots and the packed placement field |
| [`restriction_space.h`](restriction_space.h.md) | The two kinds of movement restriction and the six restrictor settings |
| [`ai_sounds.h`](ai_sounds.h.md) | The bitfield describing a sound *as the AI hears it* |
| [`smart_cast.h`](smart_cast.h.md) · [`.cpp`](smart_cast.cpp.md) · [`_impl0.h`](smart_cast_impl0.h.md) · [`_impl1.h`](smart_cast_impl1.h.md) · [`_impl2.h`](smart_cast_impl2.h.md) · [`_stats.cpp`](smart_cast_stats.cpp.md) | Asking "is this record also a *Y*" cheaply, over a closed set of types known at build time |
| [`script_value.h`](script_value.h.md) · [`_inline.h`](script_value_inline.h.md) | One script-table field shadowed as an addressable cell |
| [`script_value_container.h`](script_value_container.h.md) · [`_impl.h`](script_value_container_impl.h.md) | The set of shadow cells a record owns, written back on serialization |
| [`script_value_wrapper.h`](script_value_wrapper.h.md) · [`_inline.h`](script_value_wrapper_inline.h.md) | Binding one shadow cell to one named field of one script object |
| [`script_ini_file.h`](script_ini_file.h.md) · [`.cpp`](script_ini_file.cpp.md) · [`_script.cpp`](script_ini_file_script.cpp.md) | Configuration as scripts see it: every read a diagnosable failure, plus a write surface |
| [`script_net_packet_script.cpp`](script_net_packet_script.cpp.md) | **The wire buffer exported to scripts**: every width and quantization a script record may use |
| [`script_reader_script.cpp`](script_reader_script.cpp.md) | The read-only chunked stream exported to scripts |
| [`script_token_list.h`](script_token_list.h.md) · [`.cpp`](script_token_list.cpp.md) · [`_script.cpp`](script_token_list_script.cpp.md) | A script-built name/value set, for configuration reads constrained to a vocabulary |
| [`script_rtoken_list.h`](script_rtoken_list.h.md) · [`_inline.h`](script_rtoken_list_inline.h.md) · [`_script.cpp`](script_rtoken_list_script.cpp.md) | The same with interned names, for data-driven vocabularies |
| [`script_fvector_script.cpp`](script_fvector_script.cpp.md) | The vector, two-component vector, box and rectangle, exported to scripts |
| [`script_fmatrix_script.cpp`](script_fmatrix_script.cpp.md) | The transform, exported with its destructive operations withheld |
| [`script_fcolor_script.cpp`](script_fcolor_script.cpp.md) | The four-channel colour, exported to scripts |
| [`script_flags_script.cpp`](script_flags_script.cpp.md) | The three bit-field widths, exported — with the narrow ones' "set all" corrected |
| [`xrServer_Objects_script.cpp`](xrServer_Objects_script.cpp.md) · [`2`](xrServer_Objects_script2.cpp.md) | The base and shape records exported to scripts |
| [`xrServer_Objects_ALife_script.cpp`](xrServer_Objects_ALife_script.cpp.md) · [`2`](xrServer_Objects_ALife_script2.cpp.md) · [`3`](xrServer_Objects_ALife_script3.cpp.md) | The alife record levels exported to scripts |
| [`xrServer_Objects_ALife_Items_script.cpp`](xrServer_Objects_ALife_Items_script.cpp.md) · [`2`](xrServer_Objects_ALife_Items_script2.cpp.md) | The item records exported to scripts |
| [`xrServer_Objects_ALife_Monsters_script.cpp`](xrServer_Objects_ALife_Monsters_script.cpp.md) · [`2`](xrServer_Objects_ALife_Monsters_script2.cpp.md) · [`3`](xrServer_Objects_ALife_Monsters_script3.cpp.md) · [`4`](xrServer_Objects_ALife_Monsters_script4.cpp.md) | The creature records exported to scripts |
| [`xrServer_Objects_Alife_Smartcovers_script.cpp`](xrServer_Objects_Alife_Smartcovers_script.cpp.md) | The smart-cover record exported to scripts |
| [`xrServer_script_macroses.h`](xrServer_script_macroses.h.md) | The shared shape of every record export: the wrapper that lets a script subclass a record and override its serialization |
| [`xrEProps.h`](xrEProps.h.md) | The property-helper interface the records are written against, and the property-key path helpers |
| [`PropertiesListTypes.h`](PropertiesListTypes.h.md) | The editor's property model: typed handles onto record fields, multi-selection, revert |
| [`PropertiesListHelper.h`](PropertiesListHelper.h.md) | The concrete property factory: one call per kind |
| [`script_properties_list_helper.h`](script_properties_list_helper.h.md) · [`.cpp`](script_properties_list_helper.cpp.md) · [`_script.cpp`](script_properties_list_helper_script.cpp.md) | The same for fields that live in a script table rather than a record |
| [`ItemListTypes.h`](ItemListTypes.h.md) | One row in an editor list |
| [`pch_script.h`](pch_script.h.md) · [`.cpp`](pch_script.cpp.md) | Build-time header aggregation for the script-facing units; no decisions |

## Reading order

Start with [`alife_space.h`](alife_space.h.md) for the widths, then
[`clsid_game.h`](clsid_game.h.md) for the tags, then
[`xrServer_Object_Base.cpp`](xrServer_Object_Base.cpp.md) for the generic spawn record and
[`xrServer_Objects.h`](xrServer_Objects.h.md) for the version changelog. Only then read the
concrete record families. The registry
([`object_factory_inline.h`](object_factory_inline.h.md),
[`object_factory_register.cpp`](object_factory_register.cpp.md)) can be read at any point
and is short.
