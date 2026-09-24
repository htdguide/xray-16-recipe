# src/xrServerEntities/xrServer_script_macroses.h

> The shared shape of every record export: which methods a script subclass may override, grouped by how far down the hierarchy the record sits.

**Needs** — [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer) · [`xrEProps.h`](xrEProps.h.md) · [`xrServer_Objects_Abstract.h`](xrServer_Objects_Abstract.h.md)
**Used by** — [`game_base_script.cpp`](../xrGame/game_base_script.cpp.md) · [`game_cl_mp_script.cpp`](../xrGame/game_cl_mp_script.cpp.md) · [`game_sv_deathmatch_script.cpp`](../xrGame/game_sv_deathmatch_script.cpp.md) · [`game_sv_mp_script.cpp`](../xrGame/game_sv_mp_script.cpp.md) · [`xrServer_Objects_ALife_Items_script.cpp`](xrServer_Objects_ALife_Items_script.cpp.md) · [`xrServer_Objects_ALife_Items_script2.cpp`](xrServer_Objects_ALife_Items_script2.cpp.md) · [`xrServer_Objects_ALife_Monsters_script.cpp`](xrServer_Objects_ALife_Monsters_script.cpp.md) · [`xrServer_Objects_ALife_Monsters_script2.cpp`](xrServer_Objects_ALife_Monsters_script2.cpp.md) · [`xrServer_Objects_ALife_Monsters_script3.cpp`](xrServer_Objects_ALife_Monsters_script3.cpp.md) · [`xrServer_Objects_ALife_Monsters_script4.cpp`](xrServer_Objects_ALife_Monsters_script4.cpp.md) · [`xrServer_Objects_ALife_script.cpp`](xrServer_Objects_ALife_script.cpp.md) · [`xrServer_Objects_ALife_script2.cpp`](xrServer_Objects_ALife_script2.cpp.md) · [`xrServer_Objects_ALife_script3.cpp`](xrServer_Objects_ALife_script3.cpp.md) · [`xrServer_Objects_Alife_Smartcovers_script.cpp`](xrServer_Objects_Alife_Smartcovers_script.cpp.md) · _and 2 more_
**Tier floor** — T2: it describes an override surface, not a layout.

## Purpose

Every record export in this chapter says the same two things about its type: *here is the
type, its name and its bases*, and *here is what a script subclass of it may override*. The
second is long, it is identical for every record at the same level of the hierarchy, and
getting it wrong means a script record that silently fails to serialize.

This file names those override sets once, in **levels**, so that a record's export names its
level and inherits the whole set. It is the reason
[`object_item_script.cpp`](object_item_script.cpp.md) can offer "declare a new entity class
entirely in script" as a real feature rather than as a thing that works for trivial classes.

## The override levels

Each level adds to the one above it. A record's export names exactly one.

**Pure** — the constructor from a section name, and nothing else. A record that can be built
but does nothing.

**Abstract** — adds the two halves of save serialization (write, and read with a payload
size), the record's post-construction step, and — outside the shipping build — its
contribution to the editor's property list. **This is the level that matters**: a script
class that overrides save write and save read is a record the engine will serialize into the
same stream as a native one.

**Alife** — adds the five predicates that decide how the simulation treats the record: does
it occupy navigation-graph locations, may it be saved, may it come online, may it go
offline, is it interactive. Each is a question the alife scheduler asks and a script record
must be able to answer.

**Dynamic alife** — adds the life-cycle notifications (spawned, about to register,
registered, unregistered, switched online, switched offline) and "keep saved data anyway".
These exist only in the game build; the offline tools have no simulation to notify.

**Zone** — adds the smart-terrain surface: per-tick update, a creature entering, whether the
zone is enabled for a given creature, how suitable it is for one, registering and
unregistering a creature, handing out that creature's task, and a detection probability.
This is the level a **smart terrain** is declared at, and it is why smart terrains can be
written entirely in script: every decision a smart terrain makes is in this list.

**Creature** — adds the three allegiance accessors and the death notification.

**Monster** — adds the per-tick update on top of creature.

**Online/offline group** — adds the per-tick update and the current-task accessor.

**Item** — adds "is this still worth keeping".

**Invariants** — the level sets differ between the game build and the offline tools, and
**the difference is always a suffix**: the tools get the same set minus the simulation
hooks. A script class written against the game's surface therefore still *loads* in the
tools, with the missing overrides simply never called. That property is what lets one mod's
script records be understood by the spawn compiler.

The shipping build additionally removes the property-list override from every level, because
the editor's property model is compiled out entirely.

## Notes

**Two things are declared twice in mirror image** — once as "a script subclass may override
this" and once as "when the engine calls this, check the script side first". Both halves
have to list the same methods or a script override is accepted and never invoked. Keeping
them in one file is the entire reason this file exists; a rebuild whose binding layer derives
one from the other deletes it.

**Save and load are declared as overridable but the declarations are commented out**, along
with an alternative form that would have exposed serialization against a file stream as well
as against a packet. A script record therefore overrides the *state* serializers but not the
outer save/load bracket. No reason is recorded; the bracket writes the record's identity and
class tag, which a script record has no business rewriting, so the omission is defensible.

**The export forms come in arities** — a class with one base, two, three — and roughly half
of the combinations are commented out. The ones that remain are exactly the ones some record
uses. This is a build-time cost control, not a design statement: each unused form still costs
compilation. A rebuild generates what it needs.

**The zone level is named after zones but is used by smart terrains**, whose relationship to
an anomalous zone is only that both are restrictor volumes that notice creatures inside them.
The naming is historical and misleads on first reading.
