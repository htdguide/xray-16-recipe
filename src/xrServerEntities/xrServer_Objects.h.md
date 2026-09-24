# src/xrServerEntities/xrServer_Objects.h

> Holds the spawn-format version number and the ledger of every change ever made to it — the single most load-bearing artifact in this chapter.

**Needs** — [`xrServer_Object_Base.h`](xrServer_Object_Base.h.md) · [`ShapeData.h`](ShapeData.h.md) · [`PHNetState.h`](PHNetState.h.md) · [Data: level data — the spawn file](../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — [`Level_network_messages.cpp`](../xrGame/Level_network_messages.cpp.md) · [`PHDestroyableNotificate.cpp`](../xrGame/PHDestroyableNotificate.cpp.md) · [`Spectator.cpp`](../xrGame/Spectator.cpp.md) · [`physic_item.cpp`](../xrGame/physic_item.cpp.md) · [`server_entity_wrapper.cpp`](../xrGame/server_entity_wrapper.cpp.md) · [`server_entity_wrapper.h`](../xrGame/server_entity_wrapper.h.md) · [`stalker_animation_offsets.hpp`](../xrGame/stalker_animation_offsets.hpp.md) · [`xrServer_CL_connect.cpp`](../xrGame/xrServer_CL_connect.cpp.md) · [`xrServer_CL_disconnect.cpp`](../xrGame/xrServer_CL_disconnect.cpp.md) · [`xrServer_perform_sls_save.cpp`](../xrGame/xrServer_perform_sls_save.cpp.md) · [`xrServer_perform_transfer.cpp`](../xrGame/xrServer_perform_transfer.cpp.md) · [`xrServer_process_event.cpp`](../xrGame/xrServer_process_event.cpp.md) · [`xrServer_process_event_activate.cpp`](../xrGame/xrServer_process_event_activate.cpp.md) · [`xrServer_process_event_destroy.cpp`](../xrGame/xrServer_process_event_destroy.cpp.md) · _and 10 more_
**Tier floor** — T1: a frozen format constant and the layout history it indexes.

## Purpose

Two things live here and only one of them is code. The code declares four small records
implemented in [`xrServer_Objects.cpp`](xrServer_Objects.cpp.md). The other thing is the
**spawn version ledger**: a numbered list of every layout change to every entity record since
2002. It is the key to every version gate in the chapter — a gate reading "if version > 46"
is meaningless without the line that says what 47 added.

## State

```text
CONSTANT spawn_version = 128     # what this engine writes; what shipped data may claim
```

**Invariants** — the number is a *monotone counter over all records at once*. There is no
per-class version: adding a field to one class bumps the number for every class. A rebuild
must keep it that way, because a record only carries one version field and every class's
reader consults it. Shipped data claims values in the low 100s; anything from 1 to 128 must
parse.

### The ledger — what each version changed

Read this as the decoder ring for every `IF version > N` in this chapter. Entries are
compressed to the field or the decision; the affected class is named because the gate is in
that class's reader.

| Version | Change |
|---|---|
| 10–13 | physic object gains a fixed-bone list; hanging lamp gains spot brightness, flags, mass |
| 14–17 | the hierarchy is re-parented twice (physic object under alife object, then under dynamic object); anomalous zone inherits dynamic-object calls; records gain the visual mixin so the editor can draw them |
| 18 | hanging lamp gains a startup animation |
| 19 | teamed records stop saving health |
| 20 | vectors move from the update record into the state record |
| 21 | a global class-hierarchy rewrite — the largest single break in the ledger |
| 22–28 | anomalous zone gains artefact spawning, then its weights change from integer to real, then it gains a zone type; alife object gains the spawn identifier, group control, and its probability changes from a byte to a real |
| 29–31 | physic object gains an animation; trader gains ordered artefacts and supplies |
| 32 | **only the dynamic visual object serializes its model** — before this, several classes wrote it independently |
| 33–36 | graph point and level changer address the destination by level *name* instead of numeric identifier; level points gain fields; trader gains an organization identifier, humans gain known traders, tasks gain a try count, personal tasks cease to exist |
| 37 | the binocular stops being a weapon record and becomes a plain item |
| 38–39 | human gains equipment and weapon preferences; anomalous zone gains start power |
| 40 | physic object gains an activate flag; weapon gains an addon-state flag byte |
| 41–43 | torch gains a glow, then a guide bone; hanging lamp gains a glow texture and radius |
| 44–49 | hanging lamp gains fixed bones and health, then has properties removed; searchlight gains guide/rotation/cone bones; weapon gains the ammo-type index (47) |
| 50 | alife object gains its flag word |
| 51–57 | bolt, explosive and a condition field appear; the level changer's angle widens from one real to three (54); the car re-parents (55); the physic object gains a source identifier (57) |
| 58 | alife object gains the custom-data overlay string |
| 59–63 | PDA gains its original owner; inventory item gains a place field then loses it for a flag; physic object gains a bone mask and root bone; alife object gains the story identifier; the trader's money bug is fixed |
| 64–66 | physic object's flags, source and saved bones move up into the physics-skeleton mixin, and so does its startup animation |
| 67–68 | the custom zone and the monster/stalker bases appear; hierarchies change |
| 69 | **the object-broker serialization convention changes** — the generic container read/write used from here on; lamp and helicopter re-parent |
| 70–71 | the base record gains the script version, then the client-side opaque blob |
| 72–75 | inventory item's place becomes a flag; monsters gain in/out restrictor lists; the space restrictor becomes a class |
| 76–79 | trader gains a specific character, a climbable appears, infinite-ammo flags, anomalous zone gains three power fields |
| 80 | the spawn identifier moves from the alife object up into the base record |
| 81–85 | the spawn-group record grows, then dissolves into the base record; the alife object's probability moves up and its spawn group is removed |
| 86–87 | trader gains community index, then rank and reputation |
| 88–89 | creature gains dynamic restrictions; the actor gains its vehicle holder |
| 90–93 | PDA gains a specific character and info portion; stalker gains a demo flag; actor gains the physics-skeleton base; car health moves into the state record |
| 94 | **the client blob's length field widens from 8 to 16 bits** |
| 95 | creature gains a killer identifier |
| 96–98 | three identifier-to-string conversions: the trader's character profile, the PDA's info portion, the document's info portion |
| 99–100 | the climbable re-parents twice |
| 101–103 | the creature phantom appears; the zone owner moves from the anomalous zone down to the custom zone |
| 104 | the visual mixin gains its flag byte |
| 105–108 | trader gains a full name; custom zone gains enabled/disabled durations, then a start shift; trader loses its event list |
| 109–111 | monster base gains a special-object identifier; human abstract loses a great deal; stalker loses the demo flag |
| 112 | **the repeat-spawn control fields are removed from the base record**, and five multiplayer target classes cease to exist; the alife object gains the spawn-story identifier |
| 113 | the anomalous zone loses six artefact- and power-related fields; the custom zone loses attenuation and period |
| 114–116 | monster gains a task-reached flag; **creature health is rescaled from 0..100 to 0..1** (115); creature gains a game death time |
| 117 | level changer gains silent mode |
| 118 | the human brain loses its known-customer list |
| 119 | hanging lamp gains three volumetric-light parameters |
| 120 | smart cover gains enter and exit minimum-enemy distances |
| 121 | the game-type field becomes a 16-bit mask instead of a byte |
| 122 | weapon gains a packed grenade count; **physics update vectors stop being quantized and become full 32-bit reals** |
| 123 | inventory item gains its upgrade list |
| 124 | inventory box gains can-take/closed; trader gains the same pair for its corpse |
| 127 | climbable gains a material name |
| 128 | smart cover gains a can-fire flag |

**What the ledger tells a rebuilder.** Versions 115 and 122 are the two that change the
*meaning* of bytes rather than their presence, and both need a conversion on read, not a
skip. Version 21 is a hierarchy rewrite with no compatibility path — records below it are not
expected in shipped data. Everything else is either an append or a skip.

## Exported units

- `CSE_Shape` — the volume mixin; see [`xrServer_Objects.cpp`](xrServer_Objects.cpp.md).
- `CSE_Spectator` — the multiplayer observer: a record with no payload at all.
- `CSE_Temporary` — a record that is nothing but a navigation vertex.
- `CSE_PHSkeleton` — the mixin for "this record can remember a ragdolled pose".
- `CSE_AbstractVisual` — a base record plus a model.
- `F_entity_Create` — the section-to-record factory entry point, in
  [`xrServer_Factory.cpp`](xrServer_Factory.cpp.md).
