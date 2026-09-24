# src/xrGame/stalker_sound_data_visitor.cpp

> How one stalker's noise becomes another stalker's knowledge: hearing a comrade fight tells
> you there is something to fight.

**Needs** — [`stalker_sound_data_visitor.h`](stalker_sound_data_visitor.h.md) · [`stalker_sound_data.h`](stalker_sound_data.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`memory_manager.h`](memory_manager.h.md) · [`enemy_manager.h`](enemy_manager.h.md) · [`danger_manager.h`](danger_manager.h.md) · [`visual_memory_manager.h`](visual_memory_manager.h.md) · [`hit_memory_manager.h`](hit_memory_manager.h.md) · [`agent_manager.h`](agent_manager.h.md) · [`agent_member_manager.h`](agent_member_manager.h.md)
**Used by** — [`stalker_sound_data_visitor.h`](stalker_sound_data_visitor.h.md)
**Tier floor** — T2: a perception rule over two creatures' memory state.

## Purpose

This is the **information-propagation rule of the game's combat**, and it is four lines of
consequence hidden behind a chain of refusals. When a stalker hears another stalker, it may
acquire, for free, either that stalker's current danger or that stalker's current enemy —
without seeing anything itself.

It is what makes a firefight spread through a camp. One guard sees the player; the guard
shoots; every friendly stalker within earshot learns there is an enemy and where, and comes.
No squad broadcast, no alert radius, no scripting: it falls out of the senses system,
because a sound event already reaches exactly the listeners who should know.

The visitor belongs to the **listener**, is invoked by the emitter's payload, and mutates
only the listener's memory.

## State

```text
RECORD StalkerSoundDataVisitor
  listener : reference to Stalker      # whose memory is being written
```

## `visit(payload)`

**Contract** — given the payload of a sound this creature just heard, decide what the
creature learns from it. Mutates the listener's danger memory or visual memory; never the
emitter's. Runs on the sound event, not on a schedule.

```text
FUNCTION visit(payload)
  speaker := payload.emitter

  IF listener.memory.enemy.selected EXISTS
    RETURN                          # already fighting: I have my own problem
  IF listener.is_enemy_of(speaker)
    RETURN                          # I do not learn from people I am shooting at

  IF speaker.memory.enemy.selected IS none
    # the speaker is not fighting, but may be uneasy — inherit the unease
    IF listener.memory.danger.selected IS none
      AND speaker.memory.danger.selected EXISTS
        listener.memory.danger.add(speaker.memory.danger.selected)
    RETURN

  speaker_enemy := speaker.memory.enemy.selected
  IF speaker_enemy.being_destroyed
    RETURN
  IF NOT listener.is_enemy_of(speaker_enemy)
    RETURN                          # his enemy is my friend; I learn nothing
  IF NOT speaker.alive OR NOT listener.alive
    RETURN

  listener.memory.make_object_visible_somewhen(speaker_enemy)
```

**Invariants** — the guards are ordered by cost and by meaning, and the order is not
arbitrary:

1. **A listener already in combat learns nothing.** This is the most consequential refusal
   in the file. It stops a creature's attention being dragged from the enemy shooting at it
   to whatever a distant comrade is shouting about. It also bounds the propagation: a camp
   converts to combat in one wave, not in a loop of creatures re-alerting each other.
2. **Nothing is learned from an enemy.** The listener does not inherit a hostile speaker's
   target — which, since that target is often the listener, would otherwise be absurd.
3. **Danger is only inherited by the undisturbed.** A listener that already has its own
   danger keeps it; the rule adds unease, it does not overwrite it.
4. **An enemy is only inherited if it would be your enemy too.** Relations are pairwise, so
   the speaker's enemy is checked against the *listener's* relation, not assumed transitive.
5. **Both parties must be alive** at the moment of delivery. A sound in flight can outlive
   its emitter (the payload is built for that), and a dying man's last shout does not recruit.

**Notes** — the two outcomes are deliberately unequal in strength. The no-enemy branch
copies a *danger* — a soft, decaying record that nudges the listener's motivation weighting
and makes it wary. The enemy branch instead marks the speaker's enemy as **visible at some
point** in the listener's visual memory: a fabricated sighting, with no position of its own,
which is enough for the listener's enemy selection to pick that entity up and for the combat
branch to become reachable. The original's own diagnostic message calls this a *fiction*,
and it is: the creature acts on something it never saw. That is the correct design for this
game — the alternative, copying the speaker's remembered position, would make every stalker
in a camp shoot at the same wrong spot simultaneously.

A disabled alternative in the source copied the speaker's *hit* record instead — that is,
taught the listener where the shot came from. It was evidently too strong; the fictional
sighting leaves the listener to find the enemy itself.

## `object`

**Contract** — the listening stalker this visitor belongs to.
