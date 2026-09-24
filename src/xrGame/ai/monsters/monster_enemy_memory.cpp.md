# src/xrGame/ai/monsters/monster_enemy_memory.cpp

> The hostiles a creature currently knows about, admitted through four separate channels — sight, proximity, a recent hit, and a dangerous sound — each scored by a danger value that weighs how much it is hated against how near it is.

**Needs** — [`monster_enemy_memory.h`](monster_enemy_memory.h.md) · [`monster_enemy_manager.h`](monster_enemy_manager.h.md) · [`monster_hit_memory.h`](monster_hit_memory.h.md) · [`monster_sound_memory.h`](monster_sound_memory.h.md) · [`monster_home.h`](monster_home.h.md) · [`basemonster/base_monster.h`](basemonster/base_monster.h.md) · [`dog/dog.h`](dog/dog.h.md) · [`ai_monster_squad.h`](ai_monster_squad.h.md) · [`memory_manager.h`](../../memory_manager.h.md) · [`visual_memory_manager.h`](../../visual_memory_manager.h.md) · [`enemy_manager.h`](../../enemy_manager.h.md)
**Used by** — [`monster_enemy_memory.h`](monster_enemy_memory.h.md)
**Tier floor** — T3: a keyed collection, four admission paths, a per-entry score

## Purpose

A creature's decision to fight, flee or ignore starts here. This is the set of hostiles it
knows about, with where and when each was last located and a *danger* number that ranks them.

Four things put an entity in the set, and they are genuinely different channels:

1. **Sight or proximity.** Anything the creature's general enemy perception offers is admitted
   if it is visible *or* within a tuned "feel" radius. The feel radius is the creature's blind
   awareness — it knows something hostile is right there without looking.
2. **A recent hit.** Whatever hit the creature within the last second becomes an enemy, if it
   is a legitimate target and within a separate tuned radius. This is what makes a sniped
   creature turn on its attacker.
3. **A dangerous sound.** A sound classed as dangerous within the last two seconds names its
   maker, and that maker is admitted subject to the conditions below.
4. **The player, unconditionally**, subject to distance and a mutual-visibility check.

Channels 2, 3 and 4 all carry a condition worth stating plainly, because it is surprising and
it is load-bearing: **the player must be able to see the creature** for the creature to react.
It is stated as a note in the source and flagged there as conflicting with the off-level
simulation's premises. It makes creatures unresponsive when the player is unobserved — an
optimisation or an immersion choice, recorded here as neither confirmed.

## State

```text
RECORD EnemySighting
  position : vector     # where the enemy was when last located
  vertex   : int        # the navigation vertex it occupied
  time     : int        # global clock at that location
  danger   : real       # recomputed every tick; see below

RECORD EnemyMemory
  creature  : BaseMonster
  retention : int                       # milliseconds; default 15000, overridden at bind
  entries   : map<entity, EnemySighting>
```

Invariants: every key is alive, not destroyed, on a different team from the creature, and still
considered a useful target — the eviction pass restores all four each tick. `danger` is
meaningless outside the tick that computed it.

## `update`

**Contract** — the per-tick step, in a fixed order: hit channel, sound channel, sight and
proximity, the player, evict, re-score. Called once per creature update. Requires the creature
to be alive.

```text
FUNCTION update()
  # --- channel: whatever hit me in the last second -----------------------
  IF hit_memory.has_hits() AND now < hit_memory.last_hit_time() + 1000
    attacker = hit_memory.last_hit_source()
    IF attacker is an entity
       AND creature considers it a useful enemy
       AND distance(creature, attacker) < creature.feel_recent_attacker_range()
      remember(attacker)
      IF creature is a dog THEN creature.squad().declare_home_in_danger()

  # --- channel: a dangerous sound in the last two seconds ------------------
  IF sound_memory.has_sounds()
    (sound, dangerous) = sound_memory.most_dangerous()
    IF dangerous AND now < sound.time + 2000
      maker = sound.source, if it is an entity
      IF creature considers maker a useful enemy
         AND vertical distance to the PLAYER < 10
         AND horizontal distance to the PLAYER < creature.feel_sound_maker_range()
         AND the PLAYER can see the creature
        remember(maker)
        IF creature is a dog THEN creature.squad().declare_home_in_danger()

  # --- channel: sight and proximity ---------------------------------------
  FOR EACH candidate IN creature.perceived_enemies()
    IF distance(creature, candidate) < creature.feel_enemy_range()
       OR creature.saw_recently(candidate)
      remember(candidate)

  # --- channel: the player ------------------------------------------------
  IF horizontal distance to player < creature.feel_enemy_range()
     AND vertical distance to player < 10
     AND the player is a useful enemy
     AND the player can see the creature
    remember(player)

  evict_stale()
  rescore()
```

**Notes** — in the sound channel the distances are measured **to the player**, not to the
entity that made the sound. Combined with the mutual-visibility test, the whole channel only
fires when the player is nearby and looking. The source flags this as a probable mistake or a
deliberate optimisation and does not resolve it; the shipped behaviour is what is described
here.

The vertical separation limit of ten world units appears twice and is the only thing stopping
a creature reacting to a player directly above or below it through a floor. It is not derived.

The dog special-case — a dog that is hit, or that hears a dangerous sound, tells its whole pack
that home is threatened — is the one creature-specific branch in an otherwise generic file. It
exists because dog packs are the only creatures with a shared home-defence behaviour; in a
rebuild it belongs behind a virtual "on threatened" hook on the creature rather than a type
test here.

Horizontal and full-3D distance are used inconsistently across the four channels: the hit and
sight channels use full distance, the sound and player channels use horizontal distance with a
separate vertical gate. Reproduce the inconsistency; it changes which creatures notice the
player on a staircase.

## `remember`

**Contract** — two forms. The one-argument form records the enemy's live position, navigation
vertex and the current clock, overwriting any existing entry unconditionally. The four-argument
form takes a position, vertex and time from elsewhere — another creature's report — and
overwrites an existing entry **only if the report is newer**. That asymmetry is what stops a
pack-mate's stale sighting overwriting a fresh first-hand one.

Both forms set the danger to zero; scoring happens later in the tick.

## `evict_stale`

**Contract** — removes every entry whose subject is missing, dead, being destroyed, now on the
creature's own team, no longer a useful target, or whose sighting has aged past the retention
window.

**Notes** — the team check means a creature stops hating something that changes allegiance,
mid-fight, with no transition.

## `rescore` — the danger value

**Contract** — recomputes every entry's danger from the creature's *relation* to that entity
and the distance to where it was last located.

```text
FOR EACH (enemy, sighting) IN entries
  r = creature.relation_rank(enemy)                     # a small non-negative integer
  d = straight_line(creature.position, sighting.position)
  sighting.danger = (1 + r * r * r) / (1 + d)
```

**Notes** — relation enters cubed and distance linearly, so hatred dominates proximity by a
wide margin: a strongly-hated enemy three times further away still outranks a mildly-hated
near one. That ratio is the whole character of monster target selection and is the thing to
preserve. The `1 +` on both sides keeps a zero relation and a zero distance both finite.

Distance is measured to the *remembered* position, so a creature keeps prioritising where it
last saw something.

## `best_enemy` / `best_enemy_info`

**Contract** — the highest-danger entry, but searched in two passes: **first among enemies
inside the creature's home area**, and only if none is found, among all of them. Answers
nothing when the set is empty; `best_enemy_info` answers a sighting with time zero in that
case, which is how callers detect emptiness from the info alone.

```text
FUNCTION find_best() -> optional<entry>
  best = none; highest = 0
  FOR EACH (enemy, sighting) IN entries
    IF NOT creature.home().contains(sighting.position) THEN CONTINUE
    IF sighting.danger > highest THEN highest = sighting.danger; best = entry

  IF best is none
    highest = 0
    FOR EACH (enemy, sighting) IN entries
      IF sighting.danger > highest THEN highest = sighting.danger; best = entry

  RETURN best
```

**Invariants** — the threshold starts at zero and the comparison is strict, so an entry whose
danger is exactly zero is never chosen. That cannot happen with the scoring above (the
numerator is at least one) but it does mean an entry admitted this tick and read before
`rescore` would be invisible.

**Notes** — the home-first preference is what makes a creature with a home defend it rather
than chase the nearest threat across the level. A creature with no home area treats every
position as inside it, so the first pass succeeds immediately and the fallback never runs.

## `forget_entity`

**Contract** — removes the entity from this memory and forwards the same notice to the
creature's enemy *manager*, so the selected enemy is cleared in the same call. The forwarding
is the reason this routine exists rather than a plain erase.
