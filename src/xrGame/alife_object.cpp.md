# src/xrGame/alife_object.cpp

> The game-side half of the base alife server object: turning an entity's authored "what I am carrying" text into actual spawned items.

**Needs** — [`xrServer_Objects_ALife.h`](../xrServerEntities/xrServer_Objects_ALife.h.md) · [`xrServer_Objects_ALife_Items.h`](../xrServerEntities/xrServer_Objects_ALife_Items.h.md) · [`alife_simulator.h`](alife_simulator.h.md) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: text parsing and a spawn loop; no layout, no device, no frame budget

## Purpose

Every alife server object carries an optional block of configuration text authored into
its spawn record — a small `ltx` document embedded in the entity rather than referenced
by name. This file interprets the part of that text that says what the entity should be
*holding*. It runs once, when the entity is first created, and is the reason a randomly
placed stalker in the shipped data comes with a plausible and varied kit rather than an
identical one.

It lives in the game module rather than with the server-object definitions because
spawning requires the alife simulator, the configuration database and the script engine —
none of which the entity-definition module (shared with the offline tools) is allowed to
reach. The class is declared there; only this one behaviour is compiled here.

## State

Stateless — it reads the entity's own embedded configuration and its graph position, and
emits new entities through the simulator.

The embedded text is an `ltx` document with two independent kinds of section:

```text
RECORD SpawnSuppliesConfig            # the entity's embedded ltx text
  loadout_sections : list<Section>    # "spawn_loadout", "spawn_loadout2", ... consecutive
  spawn_section    : optional<Section># "spawn"
```

A **loadout** section is a menu: exactly one line of it is chosen. The `spawn` section is
a list: every line of it is rolled independently. Both may be present, and the loadouts
are read first. The distinction is the load-bearing one — "pick one rifle from this list"
versus "roll for each of these consumables" — and it is the whole reason two mechanisms
exist rather than one.

A line in either section is `item_section = <options>`, where the options are a
comma-separated tuple whose **first element, if numeric, is a count** and whose remaining
elements are named flags and assignments:

```text
RECORD SupplyLine
  section   : text            # the item's configuration section; must exist or the line is skipped
  count     : int   = 1       # first tuple element; meaning differs, see below
  scope     : flag  = false   # attach a sight, if this weapon accepts one
  silencer  : flag  = false
  launcher  : flag  = false
  cond      : real  = 1.0     # "cond=<x>": the item's condition, 0..1
  ammo_type : int   = 0       # "ammo_type=<n>": index into the weapon's ammo_class list
  prob      : real  = 1.0     # "prob=<x>": per-unit spawn probability (spawn section only)
  level     : text            # "level=<name>": loadout lines only; see below
```

## `spawn_supplies`

**Contract** — two forms: the no-argument one uses the entity's own embedded
configuration text; the explicit one takes any text. Empty or absent text is a silent
no-op. Spawns zero or more new server objects as children of this entity, each at this
entity's position and on this entity's graph and level vertices. Not reentrant, not
thread-safe, and allocating; it runs once at entity creation, outside any frame budget.

```text
FUNCTION spawn_supplies(ini_text)
  IF ini_text is empty -> RETURN

  # The script layer gets first refusal: a mod may replace the whole mechanism.
  IF script function "ai_stalker.CSE_ALifeObject_spawn_supplies" exists
    IF it returns true when called with (this entity, entity id, ini_text)
      RETURN                         # the script did the spawning; do nothing else

  config = parse ini_text as an ltx document
  process_loadouts(config)
  process_spawn_section(config)
```

**Invariants** — the script override is checked *before* anything is parsed and takes the
whole behaviour, not part of it. A rebuild must keep that hook and its exact name: the
shipped and modded script layers rely on being able to replace loadout generation
wholesale, and a partial override (script first, then engine anyway) would double every
stalker's kit.

## Loadout sections — "choose one"

**Contract** — reads `spawn_loadout`, then `spawn_loadout2`, `spawn_loadout3` and so on,
stopping at the first index that does not exist. Each existing section contributes
exactly one chosen line. This is how a stalker gets one primary weapon *and* one pistol
*and* one set of consumables: each loadout section is a slot.

```text
FUNCTION process_loadouts(config)
  index = 0
  section_name = "spawn_loadout"
  WHILE config has section_name
    level_name = name of the level this entity's game vertex belongs to
    candidates = empty list
    FOR EACH line IN config.lines(section_name)
      IF line options contain "level="
        IF line options mention level_name
          candidates.append(line)      # level-gated: only offered on that level
      ELSE
        candidates.append(line)        # ungated: always offered

    IF candidates is not empty
      line = candidates[random_below(candidates.count)]
      IF the item section named by the line exists in the configuration database
        spawn_one_supply(line, count_means_ammo_boxes = true)

    index = index + 1
    section_name = "spawn_loadout" + index
```

**Invariants** — the level gate is matched by *substring*, against the level name of the
entity's current **game-graph** vertex, not the loaded level. An offline stalker on a map
nobody has visited still gets that map's loadout, which is the point: the kit is decided
once at creation and must reflect where the entity actually is in the world.

Enumeration stops at the first missing index, so the sections must be consecutive; a gap
silently truncates the list. A rebuild may scan for all of them instead, which is
strictly more permissive and safe against the shipped data.

**Notes** — the "candidates, then one random draw" shape is deliberately not "roll each
line in turn": it guarantees exactly one item per slot regardless of how many options the
author listed, so adding a variant to a loadout does not make the slot more likely to
fire. Draws come from the *global* random stream here, not the entity's own — meaning two
identically configured stalkers spawned in the same instant get different kits, but the
sequence is not reproducible from the entity alone.

## Spawn section — "roll for each"

**Contract** — reads the single `spawn` section and, for each line, attempts `count`
independent spawns, each gated on the line's probability.

```text
FUNCTION process_spawn_section(config)
  FOR EACH line IN config.lines("spawn")
    IF the item section does not exist in the configuration database -> CONTINUE
    count = line.count (at least 1)
    FOR i IN 1..count
      IF random_unit_interval() < line.prob
        item = alife.spawn_item(line.section, position, level_vertex, game_vertex, parent = this)
        apply_addons(item, line)
        set item condition to line.cond
```

**Invariants** — here `count` is a number of *attempts*, each independently rolled, so
`count = 3, prob = 0.5` yields between zero and three items. In a loadout section the same
field means something different (see below). That divergence is the single most
error-prone thing on this page and a rebuild should document it at the format level rather
than in the code.

An item section that is not present in the configuration database skips the line rather
than failing the spawn. That tolerance exists because the shipped data outlives the mods
that referenced it.

## `spawn_one_supply` — the loadout line's item

**Contract** — the shared tail of a loadout choice: spawn the item, attach its addons,
spawn its ammunition, set its condition.

```text
FUNCTION spawn_one_supply(line)
  item = alife.spawn_item(line.section, position, level_vertex, game_vertex, parent = this)
  apply_addons(item, line)

  IF item is a weapon
    # For a weapon, `count` means boxes of ammunition, not copies of the weapon.
    ammo_classes = configuration(line.section, "ammo_class")   # a tuple of ammo sections
    IF present
      chosen = ammo_classes[line.ammo_type]   # clamped to the last entry if out of range
      IF chosen names an existing section
        REPEAT line.count TIMES
          alife.spawn_item(chosen, position, level_vertex, game_vertex, parent = this)
  ELSE
    # For anything else, `count` means copies: one is already spawned, add the rest.
    REPEAT line.count - 1 TIMES
      alife.spawn_item(line.section, position, level_vertex, game_vertex, parent = this)

  set item condition to line.cond
```

**Invariants** — exactly one weapon comes out of a weapon loadout line no matter what
count says; a weapon always arrives with at least one box of its own ammunition without
the author listing it; and `ammo_type` indexes the weapon's own declared ammunition
list rather than naming a section, so the same loadout line works for every weapon that
declares the same number of ammunition kinds.

The index walk over `ammo_class` clamps rather than wraps or fails: an `ammo_type` past
the end leaves the last entry selected. A rebuild should clamp identically — the shipped
data contains indices that rely on it.

## `apply_addons`

**Contract** — for a weapon, sets the sight, silencer and grenade-launcher attachment
flags from the line's flags. Each is applied *only if that weapon declares the
corresponding attachment as attachable*: a weapon with a permanently integrated sight, or
none at all, ignores the flag rather than acquiring a detachable one. For anything that
is not a weapon, does nothing.

## `is_spawn_supplies_flag_set`

**Contract** — answers whether a named flag is set in a line's option text. Two spellings
are accepted: the bare name (`scope`) and an assignment (`scope=true`, or
`scope=<probability>`, which is meant to make the attachment random per spawn).

```text
FUNCTION is_flag_set(options_text, flag_name) -> bool
  IF flag_name does not appear in options_text -> RETURN false
  IF the character after the name is not "="   -> RETURN true    # bare form
  IF the assigned value is "true"              -> RETURN true
  RETURN random_unit_interval() <= parse_real(assigned value)
```

**Notes** — the bare form is the original 2007 spelling and the assignment form is a later
addition; the shipped data uses only the bare form, so a rebuild that supports only bare
flags still loads every shipped level.

Matching is by substring anywhere in the option text and the flag names are not anchored,
so a flag name that is a substring of another option's value would match spuriously. The
three flag names in use (`scope`, `silencer`, `launcher`) do not collide with anything
else in the shipped data; a rebuild parsing the tuple properly, element by element, is
both simpler and correct, and nothing depends on the loose match.

The probability branch is unreachable in the original: the comparison that is supposed to
detect the literal `true` is written against the wrong sentinel and succeeds for every
value, so `scope=0.5` behaves as `scope=true`. A rebuild should implement the intent —
`=true` is certain, any other value is a probability — since nothing shipped exercises the
path either way.

## `keep_saved_data_anyway`

**Contract** — false for the base alife object. It is the hook by which a subclass
declares that its saved state must be preserved even when the save/load machinery would
otherwise consider the entity's data disposable. Overriding it is how a few entity kinds
survive a reload that others do not; the base answer is "no special treatment".
