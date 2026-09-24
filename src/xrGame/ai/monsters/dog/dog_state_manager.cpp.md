# src/xrGame/ai/monsters/dog/dog_state_manager.cpp

> The dog's brain: the pack version of the standard priority selector, with a squad-wide "our
> territory is under threat" latch that converts fear into aggression, and hand-offs for the
> numbered-animation machine and the dragged corpse.

**Needs** — [`dog_state_manager.h`](dog_state_manager.h.md) · [`dog.h`](dog.h.md) · [`../monster_state_manager.h`](../monster_state_manager.h.md) · [`../ai_monster_squad.h`](../ai_monster_squad.h.md) · [`../ai_monster_squad_manager.h`](../ai_monster_squad_manager.h.md) · [`../monster_home.h`](../monster_home.h.md) · [`../group_states/group_state_attack.h`](../group_states/group_state_attack.h.md) · [`../group_states/group_state_rest.h`](../group_states/group_state_rest.h.md) · [`../group_states/group_state_eat.h`](../group_states/group_state_eat.h.md) · [`../group_states/group_state_panic.h`](../group_states/group_state_panic.h.md) · [`../group_states/group_state_hear_danger_sound.h`](../group_states/group_state_hear_danger_sound.h.md) · [`../states/monster_state_controlled.h`](../states/monster_state_controlled.h.md) · [`../states/monster_state_help_sound.h`](../states/monster_state_help_sound.h.md) · [`../states/monster_state_hitted.h`](../states/monster_state_hitted.h.md) · [`../states/monster_state_hear_int_sound.h`](../states/monster_state_hear_int_sound.h.md)
**Used by** — [`dog_state_manager.h`](dog_state_manager.h.md)
**Tier floor** — T3: a priority selector plus two hand-off rules, once per creature update

## Purpose

The dog's answer to "which global state am I in". It differs from the solitary creatures' brains
(see [`../flesh/flesh_state_manager.cpp`](../flesh/flesh_state_manager.cpp.md)) in three ways, and
those three ways are the whole content of the file:

1. **Five of its nine states are the pack versions**, which coordinate through the squad.
2. **An enemy is not automatically a reason to act.** A dog reacts to an enemy only when the
   pack's territory is threatened, which is a *squad-wide latch* rather than a per-dog fact.
3. **Leaving a state has to unwind two things** the dog holds: a playing vocabulary animation and
   a physically captured corpse.

## State

The manager owns no data. Its nine registered global states:

| Global state | Implementation |
|---|---|
| rest | [pack rest](../group_states/group_state_rest_inline.h.md) |
| panic | [pack panic](../group_states/group_state_panic_inline.h.md) |
| attack | [pack attack](../group_states/group_state_attack_inline.h.md) |
| eat | [pack eat](../group_states/group_state_eat_inline.h.md) |
| heard a dangerous sound | [pack version](../group_states/group_state_hear_danger_sound_inline.h.md) |
| heard an interesting sound | generic |
| heard a call for help | generic |
| was hit | generic |
| under another creature's control | generic |

## `execute`

**Contract** — choose one global state, switch into it, run the two hand-off rules if the state
changed, execute it, and record it as the previous state. There is one early return: if the brain
would otherwise choose to idle while a vocabulary animation is still playing, it returns without
selecting anything at all, leaving the previous state active and the animation undisturbed.

**Invariants** — the previous-state record is updated after execution, as everywhere in this
chapter. The corpse capture is released on *every* transition out of eating, which is what stops a
dog from dragging a body around while fleeing.

```text
FUNCTION execute()
  squad = my_squad()
  enemy = known_enemy()

  # --- the territorial latch: three ways to decide the home region is threatened
  attack = false
  IF enemy EXISTS
    IF squad EXISTS
      IF home.enemy_is_inside_inner_region(enemy.position)   THEN squad.mark_home_in_danger()
      IF distance(self, enemy) < ATTACK_DECISION_MAXDIST     THEN squad.mark_home_in_danger()
      IF squad.home_in_danger()                              THEN attack = true
    IF home.enemy_is_inside_middle_region(enemy.position)    THEN attack = true

  IF NOT under_another_creature_control()
    IF attack
      state = (danger_rating(enemy) == strong) ? panic : attack
      IF state == panic AND squad.living_members() > 2
        state = attack                       # a big enough pack is never afraid
    ELSE IF hit_memory_is_fresh()
      IF previous_state is not was_hit AND the hit is under a second old
        squad.mark_home_in_danger()          # tell the pack where the shot came from
      state = was_hit
    ELSE IF a call for help is pending       THEN state = hear_help
    ELSE IF heard an interesting sound       THEN state = hear_interesting
    ELSE IF heard a dangerous sound          THEN state = hear_danger
    ELSE
      IF a vocabulary animation is playing   THEN RETURN      # do not interrupt an idle
      IF a corpse is available and we are hungry
        state = eat
        IF we have not claimed a corpse yet
          claim the nearest corpse and lock it against other creatures
      ELSE
        state = rest
  ELSE
    state = controlled

  switch_to(state)

  # --- hand-offs on a state change
  IF state changed AND a vocabulary animation is playing
    force_release_the_animation_channel()
  IF we were eating and no longer are AND we hold a corpse physically
    release the corpse

  active_state.execute()
  previous = state
```

## `check_eat`

**Contract** — true when the dog has a corpse to work with — either one it already claimed, or one
the corpse-memory offers — *and* the eat state itself accepts. Returns false with no further
question when neither corpse exists.

**Notes** — the claim is the important half. A dog that decides to eat immediately claims the
corpse and locks it, so the rest of the pack does not converge on the same body; the claim is
released by the eat state's exit, with a per-section timeout before anyone else may take it. The
lock is what turns a pack feeding into a queue rather than a scrum.

## Notes on the whole file

**The territorial latch is the dog's defining behaviour.** A lone enemy walking past at a distance
is *seen and ignored*: a dog only engages when the enemy has entered the pack's home region or has
come within the close-quarters threshold, and once either happens the *squad* is marked, not the
individual. Every member then reads the squad's flag and joins in. That is how a pack commits
together, and it is also why shooting at one dog from cover brings the whole pack — being hit
marks the home as threatened too, provided the hit is fresh.

The close-quarters threshold is the only tuned number in the file and it is authored **in code**,
not in a section: six world units. It is the distance at which a dog stops treating an intruder as
scenery.

**Fear is a headcount, not a stat.** The danger rating of an enemy is what normally decides between
panic and attack; the dog overrides that whenever the pack has more than two living members. The
number is authored in code and it is the single most visible piece of pack behaviour in the game:
two dogs run, three dogs charge.

**The idle-animation early return is a genuine control inversion.** Normally the brain runs on the
scheduler and always picks a state; here it declines to decide while an animation it started is
still playing, and the *animation's completion callback* runs the brain instead (see
[`dog.cpp`](dog.cpp.md)). A rebuild must keep both halves or the pack either stutters between idle
clips or cuts them off.
