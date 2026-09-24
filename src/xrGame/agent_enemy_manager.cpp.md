# src/xrGame/agent_enemy_manager.cpp

> Target assignment for a squad: pool what every member knows, decide who fights whom, swap assignments until nobody is running past a nearer target, and share the knowledge back.

**Needs** — [`agent_enemy_manager.h`](agent_enemy_manager.h.md) · [`agent_manager.h`](agent_manager.h.md) · [`agent_member_manager.h`](agent_member_manager.h.md) · [`agent_memory_manager.h`](agent_memory_manager.h.md) · [`member_enemy.h`](member_enemy.h.md) · [`member_order.h`](member_order.h.md) · [`memory_manager.h`](memory_manager.h.md) · [`visual_memory_manager.h`](visual_memory_manager.h.md) · [`sound_memory_manager.h`](sound_memory_manager.h.md) · [`hit_memory_manager.h`](hit_memory_manager.h.md) · [`enemy_manager.h`](enemy_manager.h.md) · [`ef_storage.h`](ef_storage.h.md) · [`ef_pattern.h`](ef_pattern.h.md) · [`ai_space.h`](ai_space.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md)
**Used by** — reached through its declarations in [`agent_enemy_manager.h`](agent_enemy_manager.h.md); callers name that, not this file.
**Tier floor** — T2: repeated pairwise scoring over squad-sized sets, on the AI budget

## Purpose

The largest and most consequential of the squad brain's managers. A squad in a firefight
must behave as a squad: members must not all shoot the same target while another enemy
flanks them, must not cross each other's paths to reach targets, and must share what each
of them has seen. This file decides all of that.

The shape of the solution is worth stating before the details, because it is the thing to
carry into a rebuild:

1. **Pool.** Every combat member's known enemies are merged into one list. Each entry
   records *which members know about it* as a bit per member.
2. **Rank.** Each enemy is scored by how dangerous it is — the highest probability that it
   defeats any one of our members — and the list is sorted most dangerous first.
3. **Assign greedily.** Walk the ranked list; for each enemy pick the unassigned member
   most likely to beat it. After assigning, discount that enemy's remaining danger by the
   assigned member's effectiveness and re-rank it, so that a dangerous enemy already being
   handled stops competing for the next member.
4. **Permutate.** Repeatedly swap two members' targets whenever the swap shortens the
   distance one of them must travel *and* leaves both members' effectiveness essentially
   unchanged. This is the step that stops squads from visibly crossing over.
5. **Share.** Push the pooled knowledge back into every member's own memory, so a member
   who never saw an enemy still knows where it is.

Wounded enemies are a separate mode with its own assignment, because finishing off a
wounded enemy is a different behaviour from fighting one.

## State

```text
RECORD EnemyEntry
  object          : creature
  known_mask      : squad mask   # which members know about this enemy
  assigned_mask   : squad mask   # which members are assigned to fight it
  probability     : real         # danger: best chance this enemy has against any of ours
  position, time  : vector, int  # the pooled best-known sighting

RECORD WoundedAssignment
  enemy      : creature
  processor  : entity id     # the member assigned to finish it
  processed  : bool          # it has been dealt with

RECORD EnemyManager
  enemies           : list<EnemyEntry>          # rebuilt from scratch every distribution
  wounded           : list<WoundedAssignment>   # persists across distributions
  only_wounded_left : bool
  any_wounded       : bool
  squad             : agent
```

Invariants:

- A **squad mask** is one bit per squad member. Every "which members" question in this file
  is a bit set, and the file leans on bit tricks throughout: isolating the lowest set bit
  to iterate members, and counting set bits to ask how many members are covered. A rebuild
  with a set type writes the same logic legibly, and the bit tricks become nothing.
- The enemy list is *derived*: cleared and rebuilt on every distribution. Nothing in it
  survives between passes. The wounded list is the opposite — it persists, because an
  assignment to finish off a wounded enemy must not be re-decided every frame.
- The pooled sighting for each enemy is the squad's shared belief, and it is pushed back
  into each member: a member's own memory is *widened* by the squad's.

## `distribute_enemies`

**Contract** — the entry point, run once per squad decision. Does nothing if no member is
in combat. Pools the enemies; if there are none, stops. Then takes one of two branches —
the wounded-only branch or the ordinary fight branch — and finally shares the pooled
knowledge back to every member.

```text
FUNCTION distribute_enemies()
  IF no member is in combat THEN RETURN
  pool_enemies()
  IF enemies is empty THEN RETURN
  IF only_wounded_left THEN
    assign_wounded()
  ELSE
    rank_by_danger()
    assign_greedily()
    permutate_for_distance()
  share_knowledge_back()
```

## `fill_enemies` — pooling

**Contract** — rebuilds the enemy list from every combat member's own memory. Each member
contributes its known enemies; an enemy already in the list gains that member's bit rather
than a duplicate entry. Also resets every member's effectiveness to one (meaning
unassigned), prunes stale wounded assignments, classifies the pool, and pushes each
enemy's pooled sighting into the squad's shared memory.

**Invariants** — the classification at the end drives everything downstream:

```text
FUNCTION pool_enemies()
  enemies.clear()
  FOR EACH member IN squad.combat_members
    member.effectiveness = 1                    # 1 marks "not yet assigned"
    FOR EACH enemy known to member
      add or widen the entry, setting member's bit in known_mask

  IF enemies is empty THEN RETURN

  drop every wounded assignment whose enemy is no longer in the pool

  only_wounded_left = every pooled enemy is a wounded creature
  any_wounded       = at least one pooled enemy is wounded
  FOR EACH entry -> publish its sighting into the squad's shared memory

  IF any_wounded AND NOT only_wounded_left THEN
    remove every wounded enemy from the pool     # healthy enemies first
```

- **A wounded enemy is ignored entirely while any healthy enemy remains.** Only when the
  pool is nothing but wounded enemies does the squad switch to finishing them off. This is
  the squad's priority rule and it is the reason for both flags.
- Resetting each member's effectiveness to one before assignment is what lets the greedy
  step recognize an unassigned member: an exact value of one means "free".

## `evaluate`

**Contract** — the pairwise combat estimate: given two living entities, the probability the
first defeats the second, in the unit interval. It is not computed here — it is an
**authored evaluation function** looked up in a data-driven pattern store, which the game
data supplies. The call fills the store's two operand slots and reads the pattern's value,
scaling from a percentage.

**Invariants** — the operand order is load-bearing and is used in *both* directions in
this file: ranking asks "how likely is this enemy to beat our best member", assignment asks
"how likely is this member to beat this enemy". Getting the order wrong inverts the squad's
entire target priority. A rebuild should name the two directions rather than relying on
argument order.

**Notes** — the evaluation store is a single global with mutable operand slots, filled
immediately before each read. It is not reentrant and cannot be evaluated concurrently. A
rebuild should pass the operands.

## `compute_enemy_danger` — ranking

**Contract** — scores each pooled enemy as the *maximum* over combat members of that
enemy's chance against that member, then sorts the list so the most dangerous enemy comes
first.

**Invariants** — the maximum, not the average: an enemy that would slaughter one member is
dangerous even if the rest of the squad outmatches it. Sorting descending by that score is
what makes the greedy assignment handle the worst threat first.

## `assign_enemies` — greedy assignment

**Contract** — repeatedly finds the first enemy in rank order that still has an unassigned
member who knows about it, picks the member with the best chance against it, assigns them,
and discounts the enemy's remaining danger. Stops when no assignment is possible.

```text
FUNCTION assign_greedily()
  LOOP
    chosen_member = none
    FOR EACH entry IN enemies, in rank order
      best = -1
      FOR EACH member bit IN entry.known_mask
        IF member.effectiveness != 1 THEN CONTINUE       # already assigned
        value = probability(member beats entry.object)
        IF value > best THEN best = value ; chosen_member = member
      IF chosen_member found THEN BREAK                  # take the most dangerous enemy first
    IF no chosen_member THEN BREAK

    entry.assigned_mask.set(chosen_member)
    chosen_member.effectiveness = best
    entry.probability = entry.probability * (1 - best)   # residual danger
    re-place entry in the sorted list by bubbling it down one position at a time
```

**Invariants**

- The residual discount is the heart of the algorithm. After a member is assigned, the
  enemy's danger is multiplied by the member's chance of *failing*. A dangerous enemy with
  a strong member on it drops far down the ranking, so the next member goes to the next
  real threat instead of piling on. Two members are assigned to one enemy only when its
  residual danger is still the highest in the list.
- Re-ranking is done by moving the just-discounted entry *downward* one step at a time
  rather than re-sorting. It is correct because exactly one entry changed and it can only
  have got less dangerous, so the rest of the order is intact. This is the file's one real
  performance decision and it is the right one.
- A member is assigned at most once — that is what the effectiveness sentinel enforces —
  so the loop terminates after at most one round per combat member.

## `permutate_enemies` — swapping for distance

**Contract** — after the greedy assignment, repeatedly exchange two members' targets when
doing so shortens the distance the first member must travel, the second member is farther
from that target than the first, and the exchange leaves both members' *effectiveness*
essentially unchanged. Runs until no exchange is found. Also, at the end, credits every
member with any enemy it can currently see or has just been hit by.

```text
FUNCTION permutate_for_distance()
  FOR EACH member IN squad.combat_members
    member.candidate_enemies = the indices whose known_mask includes member
    member.selected = the index whose assigned_mask includes member, if any
    member.settled = (no target was assigned)          # nothing to improve

  REPEAT
    changed = false
    FOR EACH member NOT settled
      best = distance(member, its current target)
      FOR EACH candidate IN member.candidate_enemies, other than the current one
        d = distance(member, candidate)
        IF d >= best THEN CONTINUE
        FOR EACH other member currently assigned to that candidate
          IF other cannot take member's current target THEN CONTINUE
          IF distance(other, candidate) <= d THEN CONTINUE     # other is nearer; leave it
          IF exchanging would change either member's effectiveness THEN CONTINUE
          exchange the two targets
          best = d ; found = true ; BREAK
      IF NOT found THEN member.settled = true ELSE changed = true
  UNTIL NOT changed
```

**Invariants**

- The effectiveness guard is what keeps the permutation from undoing the greedy step: a
  swap is only allowed when both members are *equally* effective against each other's
  targets, to within a float comparison. So distance is used purely as a tie-break among
  combat-equivalent assignments. This ordering of concerns — capability first, geometry
  second — is the transferable decision.
- A member with no assigned target is marked settled rather than left to churn. It will
  acquire one on its own next update, once the knowledge-sharing step has told it the
  enemy exists. (That is exactly the case the wounded assignment's giving-up path relies
  on; see below.)
- The final crediting pass gives a member's bit to every enemy it can *see now* or that
  last hit it, over and above the assignment. A member always fights what is shooting at
  it, whatever the squad decided. It is skipped in the wounded-only mode.

## `assign_wounded`

**Contract** — the other branch: every remaining enemy is wounded, and the squad must
decide who walks over to finish each one. Existing assignments are honoured where they
still make sense; the rest are filled by nearest-first, and the pass ends when every combat
member has something to do.

```text
FUNCTION assign_wounded()
  previous = the current wounded assignments ; clear them

  FOR EACH (enemy, member) IN previous
    KEEP the assignment only if ALL of:
      the enemy is still in the pool
      the member still exists and is still in combat
      the member is within the reach distance (3 metres) of the enemy
      the enemy has no other processor yet
    on keeping: re-record it and mark the member covered

  level = 0
  WHILE fewer members are covered than there are combat members
    search, over two attempts, for the (enemy, uncovered member) pair at least distance,
      considering only enemies with at most `level` members already on them
    IF nothing found on the first attempt THEN level = level + 1 and try again
    IF still nothing THEN RETURN            # see the note below
    record the assignment and mark the member covered
```

**Invariants**

- The reach distance for keeping an existing assignment is three metres — the same
  constant the individual creature's "have I reached my wounded enemy" test uses, shared
  by declaration so the two cannot drift.
- The escalating level is how the pass shares wounded enemies when there are fewer of them
  than members: the first attempt only considers enemies nobody is on, and each failed
  attempt permits one more member per enemy. Everyone gets an assignment even with a single
  wounded enemy and four members.
- Giving up when no pair is found is a *documented legitimate case*, not an error: a
  member whose only known enemy just went offline knows about no enemy at all at this
  instant, so nothing can be assigned to it. The knowledge-sharing step that runs
  immediately after will tell it about the squad's other enemies, and it will pick one on
  its next update. A rebuild must keep this path survivable rather than asserting.

**Notes** — the previous assignments are copied to stack scratch before the list is
cleared, to avoid allocating. In a rebuild that is a local list and the trick disappears.

## `assign_enemy_masks` — sharing knowledge back

**Contract** — the final step of every distribution. First, every pooled enemy is made
"visible at some point" in every combat member's memory, so that a member who has never
seen it can still be ordered to fight it. Then, for each enemy, the squad's assignment mask
is merged into the corresponding entry of the squad's shared visual, sound and hit memory.

**Invariants** — this is what turns a squad from a set of individuals into a unit: after it
runs, every member's own enemy selection can pick a target it never personally detected.
The engine's individual memory managers then behave as though the member had sensed it.

## `exchange_enemies`

**Contract** — swaps two members' assigned targets, updating both enemies' assignment masks
and both members' selections. Used only by the permutation step.

## Wounded bookkeeping

**Contract** — four small operations over the persistent wounded list: record which member
is finishing which wounded enemy (rejecting a duplicate), read that back (a sentinel means
none), mark a wounded enemy as dealt with, and read that flag.

## `assigned_wounded` / `useful_enemy`

**Contract** — the queries individual creatures ask of the squad. *Assigned wounded* asks
whether this member is the one assigned to finish this wounded enemy. *Useful enemy* asks
whether this member should engage a given enemy at all — and answers **yes** for any member
not registered in combat, and yes for any enemy the squad has not pooled. Only a member
that is part of the squad's combat assignment is constrained by it.

**Invariants** — defaulting to *yes* is the right failure mode: a member the squad has no
opinion about falls back to its own judgement rather than standing idle.

## `remove_links`

**Contract** — drops wounded assignments naming a departing object, on either side of the
pair: the enemy leaving, or the member assigned to finish it.

## `update`

**Contract** — does nothing; the distribution is driven by the squad brain's decision cycle,
not by a frame hook.

## Bit-population helper

**Contract** — counts the set bits in a squad mask, used to ask how many members are
covered. Implemented as the classic branch-free fold — pairwise sums, then nibbles, then
bytes — because it is called inside the wounded assignment's loop. In a rebuild this is a
standard library call or a set's size.
