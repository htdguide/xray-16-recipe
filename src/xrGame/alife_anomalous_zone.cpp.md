# src/xrGame/alife_anomalous_zone.cpp

> The server-side anomaly: it draws its own strength at spawn time and populates itself with artefacts by weighted lottery, so the artefacts a player eventually finds were decided before the level ever loaded.

**Needs** — [`xrServerEntities/xrServer_Objects_ALife_Monsters.h`](../xrServerEntities/xrServer_Objects_ALife_Monsters.h.md) · [`ai_space.h`](ai_space.h.md) · [`alife_simulator.h`](alife_simulator.h.md) · [`alife_time_manager.h`](alife_time_manager.h.md) · [`alife_spawn_registry.h`](alife_spawn_registry.h.md) · [`alife_graph_registry.h`](alife_graph_registry.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a weighted draw and a configuration read.

## Purpose

An **anomaly** is a first-class entity in the off-screen simulation, not level decoration.
Its **server object** decides two things when it is created, once, and then carries them
for the rest of the game: how strong it is, and which artefacts lie inside it. This file
holds those decisions.

It is a separate file from the rest of the server entity definitions because it is one of
the few server objects with real behaviour rather than just state.

## State

```text
RECORD ServerAnomalousZone                  # extends the server-side custom zone
  max_power                  : real    # drawn at spawn from a configured range
  hit_type                   : enum    # the damage kind it deals
  offline_interactive_radius : real    # the radius artefact placement is scaled over
  current_best_weapon        : none    # always absent; see below
  time_id                    : int     # stamped on each interaction query

# Invariant: the artefact draw happens exactly once, while the zone is offline.
#   Running it online would spawn artefacts into a level that has already been
#   populated and is being rendered.
```

## `spawn_artefacts`

**Contract** — Draws the zone's strength and creates its artefacts. Asserts the zone is
offline. Reads five configuration keys from the zone's own section. Silently does nothing
when the artefact count works out to zero; complains in the log and does nothing when a
count is configured but the artefact list is missing, because that combination is a data
authoring error rather than a valid "empty zone".

```text
FUNCTION spawn_artefacts()
  REQUIRE this zone is offline

  # 1. Strength: a uniform draw between the configured bounds, or exactly the lower
  #    bound when they coincide.
  max_power = uniform(min_start_power, max_start_power)     # defaults 0 .. 1

  # 2. How many artefacts: a uniform integer draw between the configured bounds.
  count = uniform_int(min_artefact_count, max_artefact_count)   # defaults 0 .. 0
  IF count == 0 THEN RETURN

  # 3. The candidate table is a flat list alternating section name and weight.
  #    An odd length is a data error. A named section that does not exist is
  #    dropped from the table with a warning, not treated as fatal — mods remove
  #    artefacts and the zones referencing them must still work.
  candidates = parse pairs from the "artefacts" line, skipping unknown sections

  # 4. One weighted draw per artefact, with replacement.
  FOR i IN 1 .. count
    r = uniform(0, 1)
    walk the candidates accumulating weight; take the first whose running sum exceeds r
    IF the walk ran off the end THEN skip this artefact
    spawn it (see below)
```

**Invariants** — The weights are used as a running sum compared against a draw from zero to
one, which means **the weights must sum to one** for the draw to be uniform over them. That
requirement is nowhere stated and nowhere enforced. A table summing to less than one leaves
a gap at the top of the range where the walk falls off the end and *no artefact is
spawned*, silently reducing the count. A table summing to more than one makes the later
candidates unreachable. The shipped data satisfies it; a rebuild should normalize and say
so.

Dropping unknown sections shortens the table without renormalizing, so a mod that removes
one artefact makes every remaining draw slightly more likely to fall off the end.

## Placing one artefact

**Contract** — Each drawn artefact becomes a real server object, owned by the simulation
and attributed to this zone.

```text
FOR EACH drawn artefact
  object = alife.spawn_item(section, zone position, zone level vertex, zone graph vertex)
  FAIL WITH "can't spawn artefact" IF it did not come back
  FAIL WITH "non-alife object in the spawn file" IF it is not a simulation object
  object.spawn_id       = this zone's spawn identifier   # the artefact belongs to the zone
  object.simulation_owned = true                         # the simulation, not a script,
                                                         #   is responsible for it
  spawn registry assigns the artefact a position inside the zone

  # Register the artefact on the cross-level graph. The registration rewrites the
  # object's placement, so the just-assigned position, level vertex and distance are
  # saved across the call and restored after it.
  remember (position, level vertex, distance)
  graph registry.change(object, from = this zone's graph vertex, to = the object's)
  restore (position, level vertex, distance)

  FAIL WITH "zones can only generate artefacts" IF it is not an artefact
  # Strength falls off linearly with distance from the zone's centre.
  object.anomaly_value = max_power * (1 - distance(object, zone) / interactive_radius)
```

**Invariants** — The save-and-restore around the graph registration is the load-bearing
oddity: moving an object between graph vertices normally *is* a teleport and resets its
fine placement, but here the placement was just computed deliberately and must survive. A
rebuild whose graph registration does not clobber placement drops the dance entirely; one
that keeps the coupling must keep it.

The artefact's strength is a linear ramp from the full zone strength at the centre to zero
at the interactive radius. An artefact placed outside that radius would get a negative
value, which nothing clamps — the spawn registry's placement is trusted to stay inside.

## `on_spawn`

**Contract** — Runs the base server object's spawn handling, then the artefact draw. This
is the only caller, which is what makes the draw once-per-lifetime.

## `bfActive`

**Contract** — Whether the zone should be considered by the simulation's interaction pass.

**Notes** — The name says *active* and the body answers true when the zone has *no* power
or is *not* interactive — the sense is inverted relative to the name. Callers presumably
read it as "skip this zone"; a rebuild should rename rather than preserve the confusion.

## `tpfGetBestWeapon`

**Contract** — The simulation asks every attacker what it would hit with. A zone answers
with no weapon, its maximum power as the damage, and its configured hit type; it also
stamps the current game time on itself. The zone *is* the weapon, which is why the returned
weapon is always absent.

## `tfGetActionType`

**Contract** — An anomaly's response to meeting anything is always attack. No conditions,
no group or mutual-detection logic — the parameters exist because the interaction interface
is shared with creatures.

## `tpfGetBestDetector`

**Contract** — Unreachable. An anomaly never searches for anything, so the interface method
it inherits asserts if called. A rebuild should split the interface rather than inherit a
method that cannot run.

## `keep_saved_data_anyway`

**Contract** — Always true. A zone's saved state is preserved even when the generic rule
would discard it, because the strength and artefact set were drawn once and cannot be
redrawn without changing what the player would find.
