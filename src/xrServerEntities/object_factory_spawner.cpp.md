# src/xrServerEntities/object_factory_spawner.cpp

> Builds a browsable index of everything the mounted data can spawn, and drives the developer's spawn panel from it.

**Needs** — [`object_factory.h`](object_factory.h.md) · [`object_factory_spawner.h`](object_factory_spawner.h.md) · [`ShapeData.h`](ShapeData.h.md) · [Seam: Debug overlay UI](../../SYSTEM-REQUIREMENTS.md#seam-debug-overlay-ui) · [Data: configuration](../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — reached through its declarations in [`object_factory_spawner.h`](object_factory_spawner.h.md); callers name that, not this file.
**Tier floor** — T3: a configuration scan and an immediate-mode panel. Debug-only; a rebuild may omit the whole file.

## Purpose

Two jobs. The first is the one worth keeping: **scan the entire mounted configuration once
and work out which sections are actually spawnable**, which is a non-obvious question the
game data does not answer directly. The second is the panel that presents the result, which
is debug tooling and carries no engine decisions.

The whole file is compiled out of the shipping build.

## `build_spawn_index`

**Contract** — runs once, after the configuration is parsed and the class registry is
filled. Walks every configuration section and files the spawnable ones into per-category
lists. Reads configuration only; allocates the index. Does not spawn anything.

```text
FUNCTION build_spawn_index()
  FOR EACH section IN configuration
    IF section has no name OR no "class" key THEN CONTINUE
    identifier = class identifier named by section."class"
    IF registry has no entry for identifier THEN CONTINUE   # class not compiled in
    category = detect_category(section."kind", section."weapon_class", identifier)

    IF category is a carryable item
      IF section has no inventory-grid size
        category = Unknown            # a template, not a real item
      ELSE IF section is marked a quest item
        category = ItemsQuest

    IF category is a weapon
      IF section."parent_section" names a different section
        category = Unknown            # an upgrade variant, not a distinct weapon
    ELSE IF category is a stalker squad
      category = category of the squad's first member, re-derived   # see below
    ELSE IF category is a vehicle or a physics object
      IF section has no "visual" key THEN category = Unknown

    IF category = Unknown THEN CONTINUE
    index[category].append(section)
```

**Invariants** — a section reaches the index only if the class registry has an entry for its
class identifier, so the index can never offer something the factory cannot build.

**Notes** — the four filters are the interesting content, because each encodes a fact about
the shipped data that is written nowhere else:

- **No inventory-grid size means not a real item.** The configuration is full of abstract
  parent sections that exist to be inherited from and carry a class but no icon geometry.
  The grid size is the cheapest reliable discriminator between a template and a thing.
- **A weapon whose `parent_section` is not itself is an upgrade variant.** The later games
  express weapon upgrades as derived sections; all of them share their parent's class and
  would otherwise flood the list with near-duplicates.
- **A squad is classified by its members, not by itself.** A scripted squad section names
  its members in an `npc` or `npc_random` list; the first name is looked up and *its*
  category decides whether the squad is filed under stalkers or mutants. If the member
  cannot be resolved the squad is dropped.
- **A vehicle or physics object with no visual is not placeable.** Same template problem as
  the grid size, in the population where grid size does not apply.

## `on_tool_frame`

**Contract** — draws the spawn panel: category list, section list (shown either by
configuration section name or by the localized in-game name), and a spawn target — near the
player, into the player's inventory, or at a chosen smart terrain on a chosen level.
Spawning a chosen section then happens once per requested copy.

The only part that is not presentation:

- **Which spawn path is used depends on whether the alife simulation is running.** With no
  alife simulation — or for a phantom, which is never an alife entity — the object is
  created directly in the loaded level and its spawn record is broadcast as a network
  message. With one, the object is handed to the alife simulation, which places it on the
  game graph and lets it exist offline. These are the two ways anything enters the world at
  all, and the panel choosing between them is a small mirror of what the game does.
- **An anomaly spawned this way gets a default shape.** A zone record carries its own volume
  (see [`ShapeData.h`](ShapeData.h.md)), which is normally authored in the level; a zone
  created out of thin air has none and would be an anomaly with no extent, so the tool
  assigns a sphere of radius 3 metres. The number is a tool default with no deeper meaning.
- **Into-inventory only applies to carryables.** Requesting it for a creature silently falls
  back to placing it near the player.

**Notes** — the localized-name display path reaches into the string table and the character
profiles, which is why the panel can show "Wolf" for a section called `stalker_wolf`. That
is a convenience; the section name is always available as a fallback.
