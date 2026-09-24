# src/xrGame/ai/ai_monsters_misc.cpp

> Decides, once per group per refresh period, whether a squad of humans should attack, hold or retreat, by estimating its collective odds against the enemies it can see.

**Needs** — [`ai_monsters_misc.h`](ai_monsters_misc.h.md) · [`ai_monsters_anims.h`](ai_monsters_anims.h.md) · [`ai_space.h`](../ai_space.h.md) · [`ef_storage.h`](../ef_storage.h.md) · [`ef_pattern.h`](../ef_pattern.h.md) · [`CustomMonster.h`](../CustomMonster.h.md) · [`memory_manager.h`](../memory_manager.h.md) · [`enemy_manager.h`](../enemy_manager.h.md) · [`group_hierarchy_holder.h`](../group_hierarchy_holder.h.md) · [`seniority_hierarchy_holder.h`](../seniority_hierarchy_holder.h.md) · [`agent_manager.h`](../agent_manager.h.md) · [`agent_member_manager.h`](../agent_member_manager.h.md) · [`stalker/ai_stalker.h`](stalker/ai_stalker.h.md) · [`Level.h`](../Level.h.md)
**Used by** — [`ai_monsters_anims.h`](ai_monsters_anims.h.md) · [`ai_monsters_misc.h`](ai_monsters_misc.h.md)
**Tier floor** — T3: arithmetic over two small lists and a cached decision; no layout, no device

## Purpose

Two unrelated things share this file for historical reasons: a group-level tactical vote,
and the numbered-animation loader declared in
[`ai_monsters_anims.h`](ai_monsters_anims.h.md). The split is arbitrary and a rebuild
should separate them.

The tactical vote is the interesting half. A group of humans facing a group of enemies has
to pick one collective posture — press the attack, hold ground, or break off — and it has
to pick the *same* one for everyone in the group, or half the squad advances while the
other half withdraws. The vote answers that by asking the engine's **evaluation-function**
layer for a pairwise victory probability between every group member and every visible
enemy, folding those into one collective probability, and comparing it against four
descending confidence thresholds. The first threshold it clears names the posture.

The result is cached on the group, not on the individual, and re-taken only after a
refresh period. That is the mechanism that keeps a squad coherent: everyone reads the same
stored answer between votes.

## State

Stateless in itself; the decision it produces lives on the group record
(`last_action`, `last_action_time`) it is handed through the team/squad/group hierarchy.

```text
RECORD GroupDecision          # owned by the group, written here
  last_action      : int      # index 0..4 into the caller's action codes
  last_action_time : int      # global-clock timestamp of the vote that produced it
```

**Invariants** — the vote is only re-taken when `now - last_action_time >= refresh_rate`;
between votes every member of the group must be handed the identical stored answer, which
is the whole point of storing it on the group.

## `choose_action`

**Contract** — given a group address (team, squad, group), four descending probability
thresholds, five caller-supplied action codes, and optionally the asking entity and a
radius, returns one of the five codes. Blocks on nothing, allocates a scratch member list.
Reads the group's cached answer if it is fresh. Writes the cache when it votes. Not
thread-safe: it parks its operands in a process-wide evaluation-function scratch area (see
Notes).

```text
FUNCTION choose_action(refresh_rate, p_attack, p_attack_cautious,
                       p_defend, p_defend_cautious,
                       team, squad, group,
                       code_attack, code_attack_cautious,
                       code_defend, code_defend_cautious, code_retreat,
                       asker, group_radius) -> int

  IF p_attack is zero THEN RETURN code_attack
      # a zero first threshold means "this group does not vote"; attack unconditionally

  g = group_record(team, squad, group)
  IF now - g.last_action_time < refresh_rate THEN
    RETURN code_for_index(g.last_action)        # everyone reads the same standing decision

  enemies = asker.memory.enemy.visible_objects()
  members = eligible_members(g, asker, group_radius)

  FOR EACH (threshold, index) IN [(p_attack,0), (p_attack_cautious,1),
                                  (p_defend,2), (p_defend_cautious,3)]
    IF group_can_beat(members, enemies, threshold) THEN
      g.last_action = index ; g.last_action_time = now
      RETURN code_for_index(index)

  g.last_action = 4 ; g.last_action_time = now
  RETURN code_retreat
```

**Invariants** — the four thresholds must descend: each successive test is a *weaker*
claim about the group's odds, so a non-monotone set makes later tests unreachable. Nothing
enforces this; it is the caller's configuration section that must be right.

**Notes** — the five outcomes are not five behaviours but five *action codes the caller
chose*, so the same vote serves several different behaviour tables. The pairs
(attack, cautious attack) and (defend, cautious defend) exist so that a group can commit at
two confidence levels rather than flipping between all-in and retreat.

## `eligible_members` (private step)

**Contract** — builds the list of group members the vote counts. A member counts only if it
is alive and *visible to AI* — an entity that has been removed from the AI-visible set (a
corpse, a hidden creature) must not contribute odds. With no asking entity, every such
member counts. With an asking entity, a member must also be within the given radius, and —
if both the asker and the member are humans — must be registered as *in combat* with the
group's coordination manager, or be the asker itself.

**Notes** — the combat-registration filter is what stops a squad counting the two men
asleep in the next room as reinforcements. The asker is always included even when it is not
registered, so a lone man who has just noticed an enemy still gets a decision rather than
dividing by an empty group.

## `group_can_beat`

**Contract** — true when the group, taken as a whole, clears the given confidence threshold
against the visible enemies. Consumes members and enemies in parallel and reports whether
every enemy got accounted for. Reads the pairwise victory probability from the evaluation
function; that function is data, not code (see Notes).

```text
FUNCTION group_can_beat(members, enemies, threshold) -> bool
  i = 0 ; j = 0                       # cursors into members and enemies
  WHILE i < count(members) AND j < count(enemies)
    skip over dead or non-entity members / enemies, advancing that cursor

    p = victory_probability(members[i], enemies[j])

    IF p > threshold THEN
      # this member is strong enough to take on more than one enemy: keep
      # assigning enemies to him while the joint odds hold up
      running = p
      WHILE advancing j
        running = running * victory_probability(members[i], enemies[j])
        IF running < threshold THEN advance i ; BREAK
    ELSE
      # this enemy is too strong for one member: pile members onto him while the
      # joint odds of them ALL failing stay acceptable
      running = 1 - p
      WHILE advancing i
        running = running * (1 - victory_probability(members[i], enemies[j]))
        IF running < 1 - threshold THEN advance j ; BREAK

  RETURN j >= count(enemies)          # true iff every enemy was covered
```

**Invariants** — the probability the evaluator returns is a percentage and is divided by
100 here; a rebuild that changes the evaluator's units must change this divisor with it.

**Notes** — the two branches are the same greedy assignment seen from both ends. When a
member out-classes an enemy, enemies are stacked onto him and the *product of his win
chances* must stay above the threshold. When an enemy out-classes a member, members are
stacked onto the enemy and the *product of their loss chances* must stay below the
complement — that is, at least one of them is expected to win. The loop is greedy and
order-dependent: it pairs the lists in whatever order they arrive, so a different member
ordering can produce a different verdict. That is a real property of the original's
behaviour, not an accident a rebuild may tidy away without changing how squads act.

The pairwise probability is obtained by writing the member and the enemy into a global
scratch area that the evaluation-function layer reads, then evaluating a function loaded
from data. That indirection is why this routine is not thread-safe and why it cannot be
handed its operands directly: the function is authored as a tree of data-driven terms that
address "the member" and "the enemy" by name. A rebuild should pass an explicit evaluation
context instead; nothing about the algorithm requires the globals.

The two item slots in the same scratch area are cleared before the vote because the
victory function does not use them and a stale item from an earlier query would otherwise
be readable.

## `load_numbered_motions`

**Contract** — the implementation of the numbered-animation loader whose contract is given
in [`ai_monsters_anims.h`](ai_monsters_anims.h.md). It lives here only because this file
already existed.
