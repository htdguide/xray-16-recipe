# src/xrServerEntities/alife_monster_brain.cpp

> The offline decision cycle for a creature: pick the smart terrain that wants it most, take the job that terrain hands out, and move toward it on the game graph.

**Needs** — [`alife_monster_brain.h`](alife_monster_brain.h.md) · [`xrServer_Objects_ALife_Monsters.h`](xrServer_Objects_ALife_Monsters.h.md) · [`alife_space.h`](alife_space.h.md) · [Data: configuration](../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: graph-level decisions, no byte layout, no device.

## Purpose

This is the coarse simulation of a creature that nobody is looking at. It is deliberately
tiny: one choice (which smart terrain), one delegation (what job that terrain gives me), one
movement update. Everything a creature does while *online* lives in chapter 24; everything
it does while offline is these forty lines. The asymmetry is the alife design — the world
keeps turning at almost no cost.

## State

```text
RECORD MonsterBrain
  owner              : ServerRecord            # the creature record this brain belongs to
  movement           : MovementManager         # game-graph path state, owned here
  smart_terrain      : optional<SmartTerrain>  # cache; validated against the owner's id
  last_search_time   : int (64-bit)            # game time of the last terrain search
  search_interval    : int (64-bit)            # from configuration, per creature type
  may_choose_tasks   : bool                    # script veto
```

**Invariants**

- The cached smart terrain is only valid while its identifier equals the one recorded on the
  creature; the accessor checks and re-resolves. The *identifier* is the truth (it is what
  is saved), the pointer is a convenience.
- `0xffff` in the creature's smart-terrain field means "not registered anywhere". The whole
  cycle branches on that one value.
- A creature that has just registered with a terrain has its last-search time reset to zero,
  so it will re-evaluate immediately if it is ever unregistered.

## `construct`

**Contract** — binds the brain to its record, creates the movement manager, and reads the
creature's **smart-terrain search interval** out of its configuration section as a
`hours:minutes:seconds` string, converted to game-time milliseconds. Aborts if the section
has no such key — every creature type must declare how often it reconsiders.

**Notes** — the interval is per creature *type*, authored, and it is the only rate control
on the whole offline population. It is what keeps a world of several hundred offline
creatures from re-scanning every smart terrain every tick.

## `update`

**Contract** — one offline cycle. Selects a task if the creature has none, then either
pursues the job its terrain gave it or falls into the default behaviour, then advances the
movement manager. Called by the alife scheduler at the coarse rate; `forced` bypasses the
rate limit on selection.

```text
FUNCTION update(forced : bool)
  select_task(forced)
  IF owner.smart_terrain_id != none
    task = smart_terrain().task_for(owner)
    IF task is none THEN FAIL WITH "smart terrain gave no task to a registered creature"
    movement.path_type = game_graph_path
    movement.target = task
  ELSE
    movement.path_type = no_path            # stand still
  movement.update()
```

**Notes** — "registered but given no task" is fatal and names the terrain in the message.
It means a smart terrain accepted a creature and then refused to employ it, which strands
the creature permanently; failing at the moment of the inconsistency is the only way to find
which terrain did it.

## `select_task`

**Contract** — if the creature is already registered with a terrain, does nothing. If a
script has vetoed task selection, does nothing. Otherwise, at most once per search interval
(or immediately when forced), scores every smart terrain in the world and registers the
creature with the best one.

```text
FUNCTION select_task(forced : bool)
  IF owner.smart_terrain_id != none THEN RETURN
  IF NOT may_choose_tasks THEN RETURN
  now = alife game time
  IF NOT forced AND last_search_time + search_interval > now THEN RETURN
  last_search_time = now

  best = none
  FOR EACH terrain IN all smart terrains
    IF NOT terrain.enabled_for(owner) THEN CONTINUE
    score = terrain.suitability_for(owner)
    IF score > best
      best = score
      owner.smart_terrain_id = terrain.id

  IF owner.smart_terrain_id != none
    smart_terrain().register(owner)
    last_search_time = 0                    # so a later unregister re-evaluates at once
```

**Invariants** — the search is a full linear scan over every smart terrain on every level,
which is why the interval exists. The scoring itself is the terrain's, not the brain's: a
terrain answers *whether* it will take this creature and *how much* it wants it, and the
brain only takes the maximum. That split is the whole smart-terrain design — the places
decide, not the creatures.

**Notes** — the "best" is seeded with the smallest representable value, so a terrain that
scores zero still wins over no terrain at all. Registration is unconditional once a winner
exists; the terrain has already said it would accept.

## `smart_terrain`

**Contract** — resolves the creature's recorded terrain identifier to the terrain record,
returning the cached one when the identifier still matches. Asserts the creature actually
has a terrain; callers check first.

## the lifecycle hooks

**Contract** — `on_switch_online` and `on_switch_offline` forward to the movement manager,
which is where the online/offline transition actually matters: an offline creature's
position is a game-graph vertex plus progress along an edge, and an online one's is a world
coordinate, so the movement manager has to convert in both directions. `on_register`,
`on_unregister` and `on_location_change` do nothing at this level and exist so that the
human brain can override them. `on_state_write` and `on_state_read` likewise write nothing —
a plain creature's brain has no persistent state of its own; its terrain identifier lives on
the record.

## `perform_attack` / `action_type`

**Contract** — a plain creature never initiates an offline attack and ignores everything it
meets. Both are overridden by the human brain, where the answers actually depend on faction
relations.
