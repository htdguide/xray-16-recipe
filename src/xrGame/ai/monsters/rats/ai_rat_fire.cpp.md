# src/xrGame/ai/monsters/rats/ai_rat_fire.cpp

> The bite, the flinch, the rule for judging a corpse worth eating, and the morale clock that decides whether a rat fights or runs.

**Needs** — [`ai_rat.h`](ai_rat.h.md) · [`ai_rat_impl.h`](ai_rat_impl.h.md) · [`ai_rat_space.h`](ai_rat_space.h.md) · [`../../../memory_manager.h`](../../../memory_manager.h.md) · [`../../../item_manager.h`](../../../item_manager.h.md) · [Seam: Networking transport](../../../../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: emits damage events into the authoritative record stream

## Purpose

Four unrelated things sharing a file because they all touch damage. The corpse valuation and the
morale clock are the two that decide behaviour.

## `Exec_Action` — the bite

**Contract** — called each tick with the rat's pending action. On an attack-begin action, plays
the attack sound and — if the enemy is alive and the bite cooldown has elapsed — emits a damage
event at the enemy and re-arms the cooldown. On an attack-end action, clears the in-progress
flag. Emits into the authoritative record stream, so the hit is replicated rather than applied
locally.

```text
FUNCTION Exec_Action()
  CASE pending_action OF
    attack_begin:
      play(attack_sound)
      IF enemy EXISTS AND enemy IS alive AND now - last_bite_at > hit_interval
        in_progress  = true
        last_bite_at = now
        emit_damage_event(target: enemy, from: self,
                          direction: normalize(enemy.position - my position),
                          power: hit_power, bone: root, impulse: 0, kind: wound)
      ELSE
        in_progress = false
    attack_end:
      in_progress = false
```

**Invariants**

- **The cooldown is enforced here, not in the brain.** The biting state re-asserts the attack
  action every tick and this routine throttles it, so the authored hit interval is the rat's
  true rate of damage regardless of how often its state runs.
- **The impulse is zero**, unlike every other creature's bite. A rat cannot physically move what
  it bites — a player being swarmed is damaged but never staggered, which is what makes a swarm
  survivable.
- **The damage is emitted, not applied.** It goes through the event stream so that in a
  multiplayer session the authoritative side resolves it; the local-authority check guards the
  emission.

## `HitSignal` — the flinch

**Contract** — the rat was hit. Records the world-space direction of the blow, the time, and the
attacker's position, and plays the injury sound unless the rat is already dying. Records only;
the reaction is the brain's.

**Notes** — the direction arrives in the rat's local frame and is rotated into world space here,
because the recorded direction outlives the frame the hit happened in and the rat will have
turned by the time anything reads it.

## `useful` and `evaluate` — judging a corpse

**Contract** — the two halves of the item system's question "is this worth going to, and how
much". `useful` is a yes/no admission; `evaluate` returns a cost, where **lower is better** and
a maximal value means "not food at all".

```text
FUNCTION useful(object) -> bool
  IF I am alive        RETURN false        # see Notes
  IF the item system already rejects it    RETURN false
  IF object IS NOT a living-thing type     RETURN false
  RETURN true

FUNCTION evaluate(object) -> real
  IF object IS alive                                          RETURN worst
  IF now - object.time_of_death >= eat_corpse_interval        RETURN worst
  IF object.remaining_food <= 0                               RETURN worst
  IF object.team == my team AND NOT eat_member_corpses        RETURN worst
  IF object.class == my class AND NOT cannibalism             RETURN worst
  RETURN object.remaining_food^2 * distance(me, object)
```

**Invariants**

- **The cost is food squared times distance, and lower wins** — so a rat prefers a *small*
  carcass nearby to a large one far away, and prefers an almost-eaten one to a fresh one. That
  is the reverse of what the shape suggests at a glance and is worth stating plainly: squaring
  the food makes a well-stocked corpse *less* attractive, and a nest therefore spreads itself
  across many carcasses instead of piling onto one. Whether that was the intent or a sign error
  is not recoverable, but it is the shipped behaviour and it is what makes a nest look like it
  is foraging.
- **The two dietary flags are enforced here and nowhere else.** Vision admits dead teammates
  unconditionally (see [`ai_rat_feel.cpp`](ai_rat_feel.cpp.md)); this is where a squeamish rat
  declines them.
- **The staleness window is authored**, so a corpse stops being food after a fixed time even
  with food left on it. Without that a level would accumulate permanent rat magnets.

**Notes** — `useful` returns false while the rat is *alive*, which reads backwards until you
notice the rat is asking about *itself as an item*: a live rat is not something to be eaten. The
corpse-selection path uses `evaluate` alone.

## `update_morale` — the nerve clock

**Contract** — called at the top of every think step. Clamps morale into its authored bounds,
and then at most once per authored interval moves it, by an amount and direction that depend on
what the rat is currently doing. Clamps again on the way out.

```text
FUNCTION update_morale()
  clamp(morale, min, max)
  IF now - last_morale_update <= restore_period
    RETURN
  last_morale_update = now

  CASE current_state OF
    free_active, free_passive:
      # wandering: converge on the authored normal from whichever side, never overshooting
      IF morale < normal   morale = min(morale + restore_quantum, normal)
      IF morale > normal   morale = max(morale - restore_quantum, normal)

    under_fire, retreat, attack_range, attack_melee, return_home:
      morale = morale + restore_quantum      # committed states recover in one direction only

    otherwise:
      # eating, pursuing, startled, patrolling, dead: morale is frozen
  clamp(morale, min, max)
```

**Invariants**

- **Only the wandering states converge; the committed ones drift.** A rat that is fighting or
  fleeing adds the recovery quantum without bound, so it grows steadily more willing to continue
  — which is what stops the brain from oscillating between attacking and fleeing on the morale
  threshold. Convergence to the authored normal only happens once the rat is calm again.
- **Several states freeze morale entirely**, eating and startle among them, so a rat eating in
  the open does not recover its nerve and is still frightened when it finishes.
- **The clock is a fixed interval, not a rate.** Morale therefore changes in discrete steps at
  an authored frequency, independent of how often the rat is scheduled — which matters because
  a settled rat runs several times less often than an active one and would otherwise recover
  several times slower.

**Notes** — a distance-to-home term is computed at the top and never used; two commented-out
variants beside each morale adjustment show it was once meant to scale recovery by how far the
rat had strayed from the nest. It does not, and a rat far from home recovers exactly as fast as
one at its centre.
