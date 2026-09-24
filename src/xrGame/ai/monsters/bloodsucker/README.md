# src/xrGame/ai/monsters/bloodsucker — the bloodsucker

Part of chapter 24 of [`SYSTEM-REQUIREMENTS.md`](../../../../../SYSTEM-REQUIREMENTS.md#7-build-order).
Shared machinery: the [chapter opener](../../README.md) and the [creature layer](../README.md).

The bloodsucker is the chapter's most elaborate creature and the one with the most
unreachable code in it. Both halves of that sentence matter to a rebuilder, so this page
separates what the shipped creature does from what this directory describes.

## What the shipped creature does

**It is usually invisible.** Not as an effect layered on top of behaviour but as a property
*of* behaviour: entering the stalking or feeding trees cloaks it and every exit path
uncloaks it, so visibility is a consequence of what the creature is doing. There are three
distinct degrees — fully cloaked while stalking, partly visible while closing to feed, fully
visible in a stand-up fight.

**It feeds.** The signature ability: close on the player, seize him, take his camera and
inventory away, hold, drain, and give control back on *every* exit path. A cooldown shared by
every bloodsucker in the world — held on the class, not the creature — makes it impossible
for two of them to take turns draining the player.

**Feeding outranks fear.** The brain checks the feed *before* grading the enemy's danger, so
a bloodsucker that can feed will feed even on an enemy it would otherwise flee. That single
ordering is the creature's character.

**It can take the player's camera entirely.** A separate ability moves the player's view to
the creature's head, removes his weapon, and widens the field of view as the creature runs.

**A scripted seize-and-drag** bypasses the brain completely: while one is running the brain
routes to an inert state and, in one case, deliberately selects nothing at all so that the
previously active state resumes when the animation ends.

## What this directory also contains, and cannot reach

The creature's **own attack composite is registered nowhere**. The line that would register
it is commented out beside the generic one that is used instead. Reachable only through it,
and therefore dead:

- the wounded withdrawal — vanish and circle whenever fifteen percent of health is lost;
- the back-approach — run at the enemy *arriving oriented to match his facing*, so the
  creature comes out of the cloak behind him;
- the reactive stalking loop — relocate while the enemy keeps spotting you, and go berserk
  when relocating stops helping.

This is the largest block of unreachable behaviour in the chapter, and it is unreachable by
an edit somebody made on purpose. A rebuild must choose consciously whether to enable it,
because doing so materially changes the fight.

The recipe keeps full twins for all of it: it is the most complete surviving statement of
what this creature's combat was meant to be, and none of it is duplicated elsewhere.

## What could not be recovered

- **Why the creature's own attack composite was disabled.** Nothing in the source says, and
  the code is complete enough to suggest it worked.
- The fifteen-second camp relocation interval, the fifteen-percent wound threshold, the
  three-second circling burst and the one-second behaviour-flip lockout are bare constants.
- The feeding screen effect oscillates exactly twice whatever the feed's length; nothing
  derives the count. Its normalisation term simplifies to dividing by one, so the oscillation
  spans the whole effect rather than the middle portion it appears to be reaching for.
- The camera drag's "ideal distance" of three tenths of a unit and its ten-degree wobble
  bound are fixed in code. A disabled triangular displacement profile sits beside the
  semicircular one that shipped.
- The drag jump names a level-specific animation and bone as string literals in the engine —
  the one place in the chapter where a *level's* content is named from code.

## Twins

| Twin | Role |
|---|---|
| [`bloodsucker.cpp`](bloodsucker.cpp.md) | The bloodsucker: how it becomes invisible, when it lets itself be seen, what it wants from the player, and how it takes his camera. |
| [`bloodsucker.h`](bloodsucker.h.md) | Declares the bloodsucker: the creature base plus invisibility, a life-draining grab, and the ability to take the player's camera away entirely. |
| [`bloodsucker_alien.cpp`](bloodsucker_alien.cpp.md) | The camera takeover: the player's view is moved to the bloodsucker's head, his weapon is taken away, and the field of view widens as the creature runs. |
| [`bloodsucker_alien.h`](bloodsucker_alien.h.md) | Declares the camera takeover: while it is active the player sees through the bloodsucker's eyes and cannot fight back. |
| [`bloodsucker_attack_state.h`](bloodsucker_attack_state.h.md) | Declares the bloodsucker's own attack composite — the one that would weave feeding and cloaked withdrawal into the shared attack tree — together with the "get behind him" approach state it owns. Nothing registers it. |
| [`bloodsucker_attack_state_hide.h`](bloodsucker_attack_state_hide.h.md) | Declares the two-step mid-combat withdrawal: run to a reserved covered spot, then stalk from it. |
| [`bloodsucker_attack_state_hide_inline.h`](bloodsucker_attack_state_hide_inline.h.md) | Breaking contact mid-fight: cloak, claim one covered spot so no packmate takes it, sprint there, and switch to stalking once arrived. |
| [`bloodsucker_attack_state_inline.h`](bloodsucker_attack_state_inline.h.md) | The combat the bloodsucker was designed to fight: feed when the chance comes, vanish and circle whenever a chunk of health is lost, and close on the enemy's back rather than his front. Unreachable in the shipped build. |
| [`bloodsucker_predator.h`](bloodsucker_predator.h.md) | Declares the full stalking loop a bloodsucker falls into after feeding: take cover, face the open ground, and wait. |
| [`bloodsucker_predator_inline.h`](bloodsucker_predator_inline.h.md) | The creature stops being a fighter and becomes an ambush: cloak, claim a covered spot, run to it, turn to face the most exposed direction, then stand perfectly still until something changes — and pick a new spot every fifteen seconds if nothing does. |
| [`bloodsucker_predator_lite.h`](bloodsucker_predator_lite.h.md) | Declares the reactive stalking loop — the same three nodes as the full predator, but re-selected each cycle from whether the enemy can currently see the creature. |
| [`bloodsucker_predator_lite_inline.h`](bloodsucker_predator_lite_inline.h.md) | Stalking that reacts: as long as the enemy keeps spotting the creature it keeps relocating, and being spotted twice in a row makes it give up hiding and charge. |
| [`bloodsucker_script.cpp`](bloodsucker_script.cpp.md) | The bloodsucker's script surface: one method. |
| [`bloodsucker_state_capture_jump.h`](bloodsucker_state_capture_jump.h.md) | Declares the state a bloodsucker parks in while a scripted seize-and-leap plays out. |
| [`bloodsucker_state_capture_jump_inline.h`](bloodsucker_state_capture_jump_inline.h.md) | Where the brain goes to get out of the way: while the scripted seize-and-leap owns the creature, its behaviour is one node that stands still and does nothing else. |
| [`bloodsucker_state_manager.cpp`](bloodsucker_state_manager.cpp.md) | The bloodsucker's brain: nine registered states, a selector that puts feeding above everything else a live enemy could provoke, and a hard bypass that hands the creature to a scripted seize whenever one is running. |
| [`bloodsucker_state_manager.h`](bloodsucker_state_manager.h.md) | Declares the bloodsucker's brain: the shared creature brain plus a feed test and a seize hook. |
| [`bloodsucker_vampire.h`](bloodsucker_vampire.h.md) | Declares the vampire behaviour: the four-node tree that carries a bloodsucker from wanting a feed to being gone again. |
| [`bloodsucker_vampire_approach.h`](bloodsucker_vampire_approach.h.md) | Declares the run-in that opens a feed. |
| [`bloodsucker_vampire_approach_inline.h`](bloodsucker_vampire_approach_inline.h.md) | Close the distance to feed: run flat out at the victim's navigation cell, ignoring cover entirely, and keep re-aiming at it as he moves. |
| [`bloodsucker_vampire_effector.cpp`](bloodsucker_vampire_effector.cpp.md) | What being fed on looks like: the screen pulses twice while the picture desaturates, and the camera is dragged to within a hand's breadth of the creature's face and held there, shivering. |
| [`bloodsucker_vampire_effector.h`](bloodsucker_vampire_effector.h.md) | Declares the two screen effects that play on the victim while a bloodsucker feeds: a post-process pulse and a camera drag toward the creature's face. |
| [`bloodsucker_vampire_execute.h`](bloodsucker_vampire_execute.h.md) | Declares the leaf state that performs the feed itself: seize the player, hold, drain, release. |
| [`bloodsucker_vampire_execute_inline.h`](bloodsucker_vampire_execute_inline.h.md) | The feed: take the player's camera and inventory away, hold for a fixed time, land the drain, give control back — and give it back on *every* exit path. |
| [`bloodsucker_vampire_hide.h`](bloodsucker_vampire_hide.h.md) | Declares the two-step retreat a bloodsucker performs after feeding: bolt, then go back to stalking. |
| [`bloodsucker_vampire_hide_inline.h`](bloodsucker_vampire_hide_inline.h.md) | After a feed the creature sprints away from where the player is, and only once it has broken contact does it resume stalking. |
| [`bloodsucker_vampire_inline.h`](bloodsucker_vampire_inline.h.md) | The whole vampire behaviour as one tree: close on the player half-cloaked, feed if the chance comes, then run and vanish — and a global cooldown so no two bloodsuckers in the world can chain feeds. |
