# src/xrGame/memory_manager.cpp

> A creature's whole memory: three senses feeding three judgements, the fixed order in which they advance each frame, and the merged answer to "what do I know about him".

**Needs** — [`memory_manager.h`](memory_manager.h.md) · [`memory_space.h`](memory_space.h.md) · [`memory_space_impl.h`](memory_space_impl.h.md) · [`visual_memory_manager.h`](visual_memory_manager.h.md) · [`sound_memory_manager.h`](sound_memory_manager.h.md) · [`hit_memory_manager.h`](hit_memory_manager.h.md) · [`enemy_manager.h`](enemy_manager.h.md) · [`item_manager.h`](item_manager.h.md) · [`danger_manager.h`](danger_manager.h.md) · [`agent_manager.h`](agent_manager.h.md) · [`agent_member_manager.h`](agent_member_manager.h.md) · [`agent_enemy_manager.h`](agent_enemy_manager.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`script_game_object.h`](script_game_object.h.md) · [`xrAICore/Navigation/level_graph.h`](../xrAICore/Navigation/level_graph.h.md) · [`xrEngine/profiler.h`](../xrEngine/profiler.h.md)
**Used by** — [`memory_manager.h`](memory_manager.h.md)
**Tier floor** — T2: a per-creature, per-frame fan-out over three memory lists

## Purpose

Every living creature owns one of these. It is the single place the three senses — sight,
sound, damage — are turned into the three things a brain actually reasons about: **who is my
enemy**, **what is worth picking up**, and **what is dangerous right now**.

The split into two layers is the design. The three *sense* managers are pure recorders: they
add and refresh and forget memories, and they know nothing about intent. The three
*judgement* managers are rebuilt from those memories every frame and are pure derivations:
they hold no history of their own. That is why a creature that goes blind still knows its
enemy — the memory outlives the perception — and why re-deriving the enemy list from scratch
each frame is cheap enough to do.

This file owns the ordering between the six, which is the part a rebuild must reproduce.

## State

```text
RECORD MemoryManager
  object   : CustomMonster      # the creature this memory belongs to
  stalker  : optional<Stalker>  # the same creature, when it is a human; none otherwise
  visual, sound, hit    : the three sense managers
  enemy, item, danger   : the three judgement managers
```

**Invariants**
- The `stalker` reference is the same object as `object` when the creature is human. Its
  presence is the test for "does this creature belong to a squad", and every squad-aware
  branch in the file keys off it. A non-human creature has no squad, no shared memory mask
  and no combat registration.
- The six sub-managers are created in the constructor and destroyed with the memory. None is
  optional; a creature with no sight still owns a sight manager that reports nothing.
- The judgement managers are cleared and rebuilt every frame. Nothing may cache a pointer
  into them across frames.

## Construction

**Contract** — creates all six sub-managers. Only the sight manager is built differently for
a human: it is given the creature in its human form, because a human's sight model consults
things — a held weapon's optics, a crouched stance — that a non-human has no notion of. The
sound manager is additionally handed the audio system's per-sound user-data visitor, which is
how an emitted sound's AI-perception attributes reach the creature at all.

## `Load` / `reinit` / `reload`

**Contract** — fan-outs. `Load` reads the creature's tuning from its configuration section
and notably **does not include the sight manager**: sight parameters are loaded later, from
the creature's own class, because a human reads them per-stance and a monster reads one set.
`reinit` and `reload` do include it.

## `update`

**Contract** — advances the whole memory by one frame. Profiled as one zone, because on a
level with many creatures this is the dominant AI cost. The order is the content.

```text
FUNCTION update(time_delta)
  visual.update(time_delta)       # 1. refresh perceptions first
  sound.update()
  hit.update()

  in_combat = stalker AND squad.member.registered_in_combat(stalker)

  enemy.reset()                   # 2. judgements are rebuilt, never accumulated
  item.reset()

  IF visual.enabled                        # 3. pour memories into the judgements
    fold(visual.objects, add_enemies = true)
  fold(sound.objects,  add_enemies = in_combat)
  fold(hit.objects,    add_enemies = in_combat)

  update_enemies(in_combat)       # 4. pick an enemy, possibly twice
  item.update()                   # 5. rank the items
  danger.update()                 # 6. age and weight the threats
```

**Notes** — three orderings here are load-bearing.

*Perceptions before judgements.* All three senses refresh before anything reads them, so a
judgement never sees a half-updated frame where sight is current and hearing is one frame
stale.

*Sight always contributes enemies; sound and damage only in combat.* Out of combat a
creature becomes hostile only to something it has **seen**. A shot heard from across the
level, or a hit from an unseen source, does not by itself produce an enemy — it produces a
*danger*, which makes the creature look around, and sight then decides. This is the rule
that stops a level-wide firefight from making every creature on the map hostile to the
player they have never seen. Once a creature *is* registered in combat, the restriction
lifts: now being shot at from cover is enough.

*Items last, danger last of all.* Item ranking depends on which objects were rejected as
enemies, and danger weighting depends on the enemy selection having been made.

## `update_enemies`

**Contract** — selects this creature's current enemy, and under one condition does the whole
selection twice.

```text
FUNCTION update_enemies(in_combat)
  enemy.update()
  IF stalker AND in_combat AND (no enemy selected OR the selected enemy is wounded)
    squad.enemy.distribute_enemies()     # the squad re-allocates targets among its members
    re-fold all three sense lists with add_enemies = true
    enemy.update()
```

**Notes** — this is the squad's target allocation, and the double pass is deliberate rather
than defensive. A squad member with **no** enemy, or one whose assigned enemy is already
down, is wasted; so the squad is asked to redistribute, and this creature then re-derives its
enemy list against the new allocation. Doing it in one pass is impossible because the
redistribution needs to know which members currently have nobody, which is only known after
the first pass.

The wounded case is the interesting half: a squad does not keep three rifles trained on a
man who is already crawling. The moment an assigned enemy is wounded, that member is freed to
be re-targeted.

## `fold` (the memory-to-judgement pass)

**Contract** — walks one sense's memory list and offers each memory to the judgements.
Filters on the memory being enabled and, for a squad member, on the memory being valid for
this member's squad bit.

```text
FUNCTION fold(memories, add_enemies)
  mask = squad bit of this creature, or 0 when not in a squad
  FOR EACH m IN memories
    IF NOT m.enabled                        THEN CONTINUE
    IF stalker AND NOT m.squad_mask.test(mask) THEN CONTINUE

    danger.add(m)                           # every memory is a potential threat signal

    IF add_enemies AND m.object IS a living entity
      IF enemy.add(m.object) THEN CONTINUE   # accepted as an enemy: not also an item

    IF stalker AND m.object IS also a human
      CONTINUE                               # humans are never scavenged by other humans
    item.add(m.object)
```

**Invariants** — an object accepted as an enemy is never also offered as an item. A creature
does not evaluate whether to pick up the man it is about to shoot.

**Notes** — a squad member reads only the memories its own squad bit is set on, even though
all the squad's memories are present in its lists. That is what lets the squad share
sightings while each member still has its own view: the bit is cleared per member when that
member's own perception of the target lapses.

The "humans are never items to other humans" rule is why a body can be looted by the player
and is ignored by the squad that killed it. Non-human creatures have no such exclusion and
do treat corpses as things of interest.

## `memory`

**Contract** — the merged answer about one object: the single most recent memory across all
three senses, with flags saying which senses contributed. Returns an empty record for a dead
creature — the dead remember nothing, which is what stops corpses from driving squad
behaviour.

```text
FUNCTION memory(object) -> MemoryInfo
  IF NOT self alive THEN RETURN empty
  newest = 0 ; result = empty
  FOR EACH sense IN (visual, sound, hit)          # in that order
    m = sense.objects find by object_id(object)
    IF m EXISTS AND m.level_time > newest
      result = m as a base memory record
      result.<sense>_info = true
      newest = m.level_time
  RETURN result
```

**Invariants** — the senses are consulted in a fixed order and each only wins if it is
*strictly* newer, so a tie between two senses in the same frame goes to sight.

**Notes** — the three flags are **not cumulative**. Only the winning sense's flag is set,
because each sense assignment overwrites the base record wholesale. A script asking "did I
both see and hear him" gets the newest sense only. This reads as a bug and is load-bearing
now: the shipped scripts were written against this behaviour.

The sight branch additionally reads the per-squad-member visibility bit and writes the result
back into the same field it read, which is a no-op. It looks like an intended *set* — "mark
this object visible to me in the merged result" — that was written as a get. Unrecovered:
nothing in the tree depends on either reading, because the merged record's visibility is not
exported to scripts (see
[`memory_space_script.cpp`](memory_space_script.cpp.md)).

The damage branch re-assigns the remembered object from the query's argument rather than from
the memory. The two agree; it is redundant.

## `memory_time` / `memory_position`

**Contract** — the same newest-across-three-senses search, returning only the timestamp or
only the remembered position. They repeat the search rather than calling `memory` because
the full merge copies a large record and both are called from inner loops of the brain's
evaluators.

All three searches are linear scans of three lists. The lists are small — a creature
remembers tens of objects, not thousands — and this is called for a specific object, so the
linearity is acceptable. It is the reason the memory lists are kept short by aggressive
forgetting rather than allowed to grow.

## `enable`

**Contract** — mutes or unmutes one object across all three senses without forgetting it.
Used to make a creature ignore something it can plainly perceive — a scripted sequence, a
friendly it must not react to. Suppression is transient: the next refresh of that memory
re-enables it (see [`memory_space_impl.h`](memory_space_impl.h.md)), so the caller must keep
asserting it.

## `remove_links`

**Contract** — the teardown notification. An object is going away and every memory holding it
must drop the reference **now**, before the object's storage is released. All six
sub-managers are told.

**Invariants** — the three sense managers are only asked when the creature is alive; the
three judgements always are. A dead creature's sense lists are not maintained and may hold
stale entries, but its judgement lists must still be clean because the danger and enemy
records outlive death for a short time while the corpse is still a signal to others.

This is the creature's participation in the engine-wide rule that a destroyed entity must be
unreferenced by every registry before its memory is released.

## `make_object_visible_somewhen`

**Contract** — injects a memory of having seen an object, without the creature having seen
it. Used by scripts and by the squad layer to make a creature "know" about an enemy. Adds a
sighting with a near-zero recognition value, then **restores the previous current-visibility
bit**, so the creature remembers the enemy without believing it can currently see them.

**Notes** — the restore is the whole point and is easy to get wrong. Injecting a sighting
naturally marks the object as currently visible; leaving that set would make the creature
aim at a target it cannot see and has no line to. The recognition value is set to the
smallest non-zero amount so that the memory exists but carries no confidence.

## `save` / `load`

**Contract** — persists the three sense lists and the danger record. The three judgement
managers are **not saved**, because they are derived and are rebuilt on the first frame
after a restore. That is the practical statement of the two-layer split.

## `on_restrictions_change` / `on_requested_spawn`

**Contract** — `on_restrictions_change` is fired when the creature's permitted movement
volumes change; only the item ranking reacts, because an item outside the creature's
permitted space is not worth wanting. The danger and enemy managers have the same hook
written and commented out — a threat does not stop being a threat because you cannot reach
it.

`on_requested_spawn` tells the three senses that an object they hold a memory of has been
asked to spawn but does not exist yet. The source marks this as a workaround for the client
spawn manager's inability to answer "will this object exist", and flags it as wanting an
architectural fix. A rebuild whose spawn path can be queried does not need the hook.
