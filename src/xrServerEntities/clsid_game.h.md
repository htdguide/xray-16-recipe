# src/xrServerEntities/clsid_game.h

> The frozen registry of class identifiers: the eight-character tags that every shipped spawn record, save game and network spawn message uses to name the type it is.

**Needs** — [`xrCore/clsid.h`](../xrCore/clsid.h.md) · [Data: level data — the spawn file](../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — [`ActorInput.cpp`](../xrGame/ActorInput.cpp.md) · [`ai_stalker_alife.cpp`](../xrGame/ai_stalker_alife.cpp.md) · [`game_cl_artefacthunt.cpp`](../xrGame/game_cl_artefacthunt.cpp.md) · [`game_cl_capture_the_artefact.cpp`](../xrGame/game_cl_capture_the_artefact.cpp.md) · [`game_cl_deathmatch.cpp`](../xrGame/game_cl_deathmatch.cpp.md) · [`game_cl_single.cpp`](../xrGame/game_cl_single.cpp.md) · [`game_cl_teamdeathmatch.cpp`](../xrGame/game_cl_teamdeathmatch.cpp.md) · [`game_sv_capture_the_artefact.cpp`](../xrGame/game_sv_capture_the_artefact.cpp.md) · [`game_sv_deathmatch.cpp`](../xrGame/game_sv_deathmatch.cpp.md) · [`game_sv_teamdeathmatch.cpp`](../xrGame/game_sv_teamdeathmatch.cpp.md) · [`object_factory_register.cpp`](object_factory_register.cpp.md) · [`object_factory_spawner.h`](object_factory_spawner.h.md) · [`xrServer_Object_Base.cpp`](xrServer_Object_Base.cpp.md) · [`xrServer_Objects_ALife_Items.cpp`](xrServer_Objects_ALife_Items.cpp.md)
**Tier floor** — T1: the identifiers are a byte-exact reinterpretation of eight ASCII characters as one 64-bit integer, and the resulting numbers are compared against values baked into shipped data.

## Purpose

An entity on disk is `(class identifier, section name, payload)`. This file is the complete
table of class identifiers the three games use. Nothing here is a decision a rebuild gets
to make: every value appears verbatim in level spawn files authored in 2006–2009, so the
table is a *transcription of an external fact*, and it is separated into its own file for
exactly that reason — it is data, not code.

The type is a 64-bit integer constructed from an eight-character ASCII tag, most significant
character first. `"AI_STL  "` — note the trailing spaces, which are significant — is the
stalker. This encoding is the load-bearing part: it makes an identifier both greppable in a
hex dump of a spawn file and cheap to compare as one machine word.

## State

```text
RECORD ClassIdentifier
  value : int (64-bit)        # the eight tag characters, char[0] in the high byte

FUNCTION tag_to_identifier(tag : text) -> ClassIdentifier
  # tag is exactly 8 characters, padded with spaces on the right
  RETURN fold of (byte(tag[i]) shifted left by 8*(7-i)) for i in 0..7
```

**Invariants**

- A tag is exactly eight characters. Shorter names are **right-padded with spaces**, and the
  padding is part of the value: `"AF_BAST "` and `"AF_BAST"` are different numbers and only
  the first is correct.
- Identifiers are unique across the whole table. The factory asserts this at registration
  (see [`object_factory_inline.h`](object_factory_inline.h.md)).
- The characters used are ASCII and positive, so the encoding never depends on the sign of a
  character type — a rebuild should read each character as an unsigned byte and be done.

## The tag families

The tags are not arbitrary strings; they are a crude namespace, and the prefix tells a reader
what kind of thing it is. A rebuild does not have to honour the convention — the values are
what matter — but a modder reading a spawn file does.

| Prefix | Family | Examples |
|---|---|---|
| `O_` | plain world objects | `O_ACTOR `, `O_HLAMP `, `O_SEARCH` (projector), `O_DOOR  ` |
| `AI_` | creatures and the graph | `AI_STL  `, `AI_FLESH`, `AI_DOG_B`, `AI_TRADE`, `AI_GRAPH` |
| `AF_` | artefacts | `AF_MBALL`, `AF_GRAVI`, `AF_BAST `, `AF_CTA  ` |
| `W_` | weapons | `W_AK74  `, `W_SVD   `, `W_KNIFE `, `W_MAGAZ ` |
| `G_` | grenades and rockets | `G_F1    `, `G_RGD5  `, `G_RPG7  `, `G_FAKE  ` |
| `EQU_` / `EQ_` | outfits and worn gear | `EQU_STLK`, `EQU_EXO `, `EQ_HLMET`, `EQ_BAKPK` |
| `II_` | inventory items | bolt, medkit, bandage, antirad, food, bottle, document |
| `Z_` / `ZS_` | zones and anomalies | `Z_MBALD `-family; the `ZS_` variants are the script-spawned twins |
| `C_` | vehicles | `C_NIVA  `, `C_HLCPTR` |
| `SV_` / `CL_` / `UI_` | the multiplayer game-rules objects | `SV_SINGL`, `CL_DM   `, `UI_AHUNT` |
| `SM_` / `WP_` / `SO_` / `DET_` | the *script* duplicates | see below |

**The `_s` shadow table.** A large set of identifiers exists twice: once as the identifier
the level editor writes into a spawn file, and once as a second identifier with a different
tag that the shipped Lua scripts use to create the same pair of classes at run time
(`"WP_AK74 "` beside `W_AK74  `, `"SM_FLESH"` beside `AI_FLESH`, `"ZS_MINCE"` beside
`Z_MINCER`). These live in [`object_factory_register.cpp`](object_factory_register.cpp.md)
rather than here, because they are not authored into level data — only scripts name them.
They exist because *Shadow of Chernobyl* shipped one naming and the later games another, and
both script sets must keep working.

## Notes

**Why this cannot be renumbered.** Every spawn record in every shipped level stores the
identifier as this 64-bit value. Re-deriving the numbers — hashing the class name, using an
enumeration index — makes every shipped level unreadable. The table is therefore an input
to the rebuild, copied character for character.

**The width is 64 bits and used to be 32.** Eight characters do not fit in 32 bits, and the
original's earlier revisions used a four-character identifier. Nothing in the shipped data
uses the narrow form any more, so a rebuild reads 64 bits and never sees the older shape.

**Identifiers appear in configuration too.** Section definitions in the `ltx` configuration
name their class with a `class` key whose value is the eight-character tag; the configuration
parser converts it with the same rule. That is how a *section* (the tuning) is bound to a
*class* (the behaviour) — see the `class` lookup in
[`object_factory_spawner.cpp`](object_factory_spawner.cpp.md).

**Dead entries.** A handful of identifiers are declared and never registered with the factory
(`ENTITY  `, `EVENT   `, `O_FLYER `, `O_DOOR  `, `O_LIFT  `, `LVLPOINT`, `AI_SPONG`,
`W_M134  `, `AI_FLE_G`'s siblings). They are remnants of cut content. A rebuild may drop
them; nothing in the shipped data references them, but keeping them costs nothing and
documents what the format once allowed.
