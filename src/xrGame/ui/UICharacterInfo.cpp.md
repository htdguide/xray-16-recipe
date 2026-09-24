# src/xrGame/ui/UICharacterInfo.cpp

> Who this person is: portrait, rank, faction, reputation and how they feel about you — read
> from the authoritative record rather than the live object, so it works for someone who is
> not loaded.

**Needs** — [`UICharacterInfo.h`](UICharacterInfo.h.md) · [`UIHelper.h`](UIHelper.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`UIInventoryUtilities.h`](UIInventoryUtilities.h.md) · [`../../xrUICore/ScrollView/UIScrollView.h`](../../xrUICore/ScrollView/UIScrollView.h.md)
**Used by** — [`UICharacterInfo.h`](UICharacterInfo.h.md)
**Tier floor** — T2: resolves server objects and polls on a frame interval

## Purpose

Five different screens show the same panel about a person. Its one structural decision governs
all of them: the panel reads the **server object** — the authoritative record — not the client
object. A character you are trading with is loaded; a character whose corpse you are looting
may not be; a character in the personal terminal's contact list is certainly not. Going to the
record makes all three the same case.

## State

See [`UICharacterInfo.h`](UICharacterInfo.h.md). Beyond the eighteen part pointers:

```text
RECORD CharacterPanel
  owner        : entity id, or absent
  force_update : bool       # refresh on the next frame regardless of the interval
  texture_name : text       # the portrait currently shown
  live_tint    : colour     # the portrait's authored tint
  dead_tint    : colour     # the tint for a dead subject
  biography    : scrolling list, optional
```

Invariant: `owner` is an identifier and may name an entity that no longer exists. Every
refresh re-resolves it and **clears the panel's subject** when the resolution fails, rather
than showing stale text.

## Resolving a character

**Contract** — One helper answers "give me the record for this entity id", and it has two
paths: through the off-screen simulation's object registry when that simulation is running,
and through the server's own entity table when it is not (a multiplayer client, or a level
without the simulation).

**Invariants** — Both paths must exist. The first is the single-player case and reaches
characters who are not loaded; the second is the multiplayer case, where there is no off-screen
simulation but there is a server-side table.

## `InitCharacterInfo`

**Contract** — Build every part from one document, each optional. The portrait is looked up
under **two element names**: the newer one, and an older one used by one game's data — and
when it is the older one, the portrait is additionally set to stretch, because that data
authors portraits at a different aspect. That is compatibility with shipped data, not a
choice.

Two tints are captured at build time: the portrait's authored tint, and a *dead* tint read
from an attribute on the portrait element with a grey default. They are the panel's only
colour state.

The three convenience forms exist because the panel is embedded three ways: with explicit
geometry, by naming a document (with a fallback document name, tried when the first is
absent), or by reading the geometry from the element that also holds the parts.

## `InitCharacter` — binding to a person

**Contract** — Point the panel at an entity and fill every part.

```text
FUNCTION init_character(id)
  owner = id
  record = resolve(id)
  info   = character info built from that record

  name       = record's character name
  rank       = rank band as localized text
  community  = the community's identifier
  reputation = reputation band as localized text

  IF a biography list exists and is enabled
    clear it; IF the character has a biography, add one wrapped text row

  show every part
  portrait = the character's portrait name
  rank icon = the rank band's identifier

  # Community icons: two textures derived from the community name by
  # suffix -- "<community>_icon" and "<community>_wide".
  IF this is NOT the player AND the community is not on the ignore list
    community icon      = community + "_icon"
    large community icon = community + "_wide"
    RETURN

  # For the player, the faction shown is the one the player has TAKEN A
  # SIDE with in a faction war, which is not the same as the player's
  # own community identifier.
  IF the player has a faction pair AND our side is not the literal "actor"
    community icon      = our side + "_icon"
    large community icon = our side + "_wide"
    RETURN

  # No faction: hide all four community icons.
  hide the community icons and their overlays
```

**Invariants**

- **Icon names are derived from the community name by suffix.** A community with no icon
  texture shows a missing texture rather than nothing, which is why the ignore list exists.
- The player's own community is the literal `"actor"` until they join a side in a faction war;
  showing that would be meaningless, so the panel substitutes the chosen side and, failing
  that, shows no faction at all. This is the panel's one piece of game knowledge.

## `InitCharacterMP`

**Contract** — The multiplayer form. Clears everything, then sets only the name and a portrait
named by the caller. Multiplayer players have no character record, no rank, no reputation and
no community, so those parts stay hidden.

## `Update` — the refresh interval

**Contract** — Called every frame; does real work **every fiftieth frame**, or immediately
after a binding.

```text
FUNCTION update()
  IF no owner THEN RETURN
  IF NOT force_update AND frame number is not a multiple of 50 THEN RETURN
  force_update = false

  IF the owner no longer exists in the simulation's registry
    owner = absent                      # the panel goes quiet rather than stale
    RETURN

  refresh the relation line
  IF the subject is a creature
    portrait tint = alive ? live tint : dead tint
```

**Invariants** — Fifty frames is the interval, and it is chosen because the two things it
refreshes — how somebody feels about you, and whether they are still alive — change on a human
timescale. A rebuild may pick a different number; what matters is that this is a **poll**,
with no change notification anywhere in the path.

The existence check goes through a registry probe rather than trusting the resolve, because
resolving a destroyed entity is not safe.

Tinting the portrait for a dead subject is the panel's only indication of death, and it is why
the two tints are captured separately at build time.

## `UpdateRelation` / `SetRelation`

**Contract** — Show how the subject feels about the player: hidden entirely when the subject
*is* the player or there is no subject, otherwise a localized band name for the goodwill,
coloured by the relation type.

**Invariants** — The goodwill and the relation type are **two separate queries** against the
relation registry, and both are directed: the registry is asked how the subject regards the
player, not the reverse. The colour comes from the relation type (enemy, neutral, friend) and
the text from the numeric goodwill, so a subject can be textually "almost friendly" while
still coloured as an enemy — which is correct, because the two are separately thresholded in
the game's own rules.

## `get_actor_community`

**Contract** — The player's faction war pair: the side they have joined and the side they
oppose. Read from a configuration section keyed by the player's community identifier, whose
value is exactly two comma-separated names. Returns false — and yields nothing — when the key
is absent, when it does not have exactly two parts, or when either part is empty.

**Invariants** — Exactly two. The faction war is a two-sided conflict and the configuration
format encodes that; a rebuild that allows more has changed the game.

## `ignore_community`

**Contract** — Is this community excluded from icon display. A membership test over a
configuration section's line names; an absent section means nothing is excluded.

**Notes** — This is how communities that have no authored icon — the ones that only exist as
game-logic groupings — avoid showing a missing texture. It is a data-driven exclusion list
precisely so that adding such a community needs no code.

## `ClearInfo` / `ResetAllStrings`

**Contract** — Blank every text field and hide every part. `ClearInfo` is what a screen calls
when its subject goes away, and is what `InitCharacterMP` runs first.
