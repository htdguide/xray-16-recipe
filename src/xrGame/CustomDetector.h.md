# src/xrGame/CustomDetector.h

> Declares the detector item, and defines in full the generic "sense a configured set of classes within a radius" list that both the artefact and the anomaly detectors are built from.

**Needs** — [`inventory_item_object.h`](inventory_item_object.h.md) · [`xrEngine/Feel_Touch.h`](../xrEngine/Feel_Touch.h.md) · [`HudSound.h`](HudSound.h.md) · [`CustomZone.h`](CustomZone.h.md) · [`Artefact.h`](Artefact.h.md) · [`ai_sounds.h`](../xrServerEntities/ai_sounds.h.md)
**Used by** — [`ActorInput.cpp`](ActorInput.cpp.md) · [`AdvancedDetector.cpp`](AdvancedDetector.cpp.md) · [`AdvancedDetector.h`](AdvancedDetector.h.md) · [`CustomDetector.cpp`](CustomDetector.cpp.md) · [`EliteDetector.cpp`](EliteDetector.cpp.md) · [`EliteDetector.h`](EliteDetector.h.md) · [`SimpleDetector.cpp`](SimpleDetector.cpp.md) · [`SimpleDetector.h`](SimpleDetector.h.md) · [`actor_communication.cpp`](actor_communication.cpp.md) · [`ai_stalker_alife.cpp`](ai_stalker_alife.cpp.md)
**Tier floor** — T2: a spatial contact set keyed by object, with per-class configuration

## Purpose

Mostly a declaration page for [`CustomDetector.cpp`](CustomDetector.cpp.md), but it also
*defines* the detect list — the reusable half of a detector — because that half is
parameterized by what it detects and the original expresses parameterization at compile
time. Two instantiations exist: one over artefacts, one over anomaly zones, and the
scientific detector carries both.

The detect list's job is to turn "these configuration sections are interesting" into "here
are the instances of them currently within a radius, each with a per-instance record I can
hang state on". It is built on the touch sense
([the glossary's *feel*](../../GLOSSARY.md)), so membership is maintained by the senses
system's enter/leave events rather than by a per-frame query.

## State

```text
RECORD DetectableType               # one per configured detectable class
  frequency_range      : pair<real, real>   # min and max click rate; unused, see below
  detect_sounds        : sound set
  zone_map_location    : text               # unused
  nightvision_particle : text               # unused

RECORD ContactInfo                  # one per instance currently in range
  type          : DetectableType    # borrowed, points into the type table
  sound_time    : real              # unused
  period        : real              # unused
  particle      : optional<particle effect>   # constructed, destroyed, never set

RECORD DetectList<K>
  types    : map<text, DetectableType>    # keyed by configuration section name
  contacts : map<K, ContactInfo>          # the live contact set
```

Invariants: every key in `contacts` has an entry in `types` — admission is gated on it, so
a contact whose type is missing is a hard failure rather than a skipped entry. Each
contact's type reference borrows from the type table, which must therefore outlive the
contact set and must not be rehashed while contacts exist.

## `DetectList::load`

**Contract** — reads a numbered family of configuration keys under a caller-supplied
prefix, stopping at the first gap. Each index contributes a section name, a frequency
range and a sound set. The prefix is what lets one section configure two independent
lists — the scientific detector declares `af_class_1…` for artefacts and `zone_class_1…`
for anomalies in the same section.

```text
FUNCTION load(section, prefix)
  i = 1
  WHILE the key "<prefix>_class_<i>" exists IN section
    name = section.read_text("<prefix>_class_<i>")
    types[name].frequency_range = section.read_pair("<prefix>_freq_<i>")
    types[name].detect_sounds   = load_sound_set(section, "<prefix>_sound_<i>_")
    i = i + 1
```

**Notes** — numbering from one and stopping at the first missing index means a
configuration cannot have a hole. That is a real constraint on the data and a rebuild must
keep it, because the shipped sections rely on it to mean "and no more".

## `DetectList::feel_touch_new` / `feel_touch_delete`

**Contract** — the senses system's enter and leave callbacks. Entering creates the
per-instance record and binds it to its type; leaving destroys it. Admission is decided
before these run, by the contact test.

## `DetectList::clear` / `destroy`

**Contract** — `clear` empties both the contact set and the senses system's own membership,
and is what the detector calls when it leaves the player's hands. `destroy` releases the
loaded sounds and is a lifetime operation, not a per-use one.

**Invariants** — clearing both halves together is the invariant that matters: emptying the
contact set alone would leave the senses system holding objects it will later try to
report leaving.

## `CAfList`

**Contract** — the artefact instantiation, plus a rank ceiling. Overrides the contact test
to reject an artefact whose rank exceeds the detector's. See
[`CustomDetector.cpp`](CustomDetector.cpp.md).

## `CZoneList`

**Contract** — the anomaly instantiation. Overrides the contact test to admit zones by
section name only; no rank applies.

## `CCustomDetector`

Declared here, implemented in [`CustomDetector.cpp`](CustomDetector.cpp.md). Exported
surface:

- `Load`, `net_Spawn` — configuration; a detector always spawns switched off.
- `ToggleDetector`, `ShowDetector`, `HideDetector` — the one entry point and its two
  directional wrappers, each a no-op unless the state machine is at rest.
- `IsWorking` — sensing enabled *and* owned by the current view entity.
- `CheckCompatibility` — may this other item be held? Holsters the detector if not.
- `OnStateSwitch`, `OnAnimationEnd` — the draw/holster state machine, advanced by
  animation completion.
- `shedule_Update`, `UpdateCL` — the sense refresh and the per-frame negotiation.
- `OnMoveToSlot`, `OnMoveToRuck`, `OnH_A_Chield`, `OnH_B_Independent` — the
  inventory-transition hooks; the last two are where the contact set is released.
- `ef_detector_type` — a constant the AI evaluation layer reads to recognize the item.
- What it demands of a subclass: `UpdateAf` (translate contacts into drawable marks),
  `CreateUI` (build the readout), `UpfateWork` (the per-frame readout refresh), and
  `ui_xml_tag` (which block of the shared layout file this model uses).
