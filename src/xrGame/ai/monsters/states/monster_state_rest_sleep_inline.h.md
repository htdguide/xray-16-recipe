# src/xrGame/ai/monsters/states/monster_state_rest_sleep_inline.h

> Sleeping: not an animation but a perception state, entered and left explicitly.

**Needs** — [`monster_state_rest_sleep.h`](monster_state_rest_sleep.h.md)
**Used by** — [`monster_state_rest_sleep.h`](monster_state_rest_sleep.h.md)
**Tier floor** — T3: two lifecycle calls and one action request

## Purpose

Twelve lines of code carrying one idea that a rebuild must not miss: **sleeping is a property of
the creature, not of the leaf.**

The leaf's execute step plays the sleeping action, which is the visible part. But its entry and
exit call the creature's own sleep and wake operations, and those do the work that matters — they
change what the creature can perceive. A sleeping creature's vision and hearing are degraded, so it
can be approached; waking restores them, and the wake also drives the getting-up animation and the
alert response.

That is why the state is not simply "play the sleep animation and stop moving". A rebuild that
implements it that way produces creatures that see you perfectly while lying down, which removes an
entire class of player approach.

## State

`Stateless.`

## `initialize`

**Contract** — run the base entry, then put the creature to sleep through its own sleep operation.

## `execute`

**Contract** — request the sleeping action and the idle voice. Nothing else — no path, no timer, no
memory access.

**Notes** — the idle voice is still played while asleep. Sleeping creatures breathe and snuffle,
which is what lets a player hear one before seeing it, and the repeat throttle comes from the
creature's own section as it does everywhere else.

## `finalize` / `critical_finalize`

**Contract** — both run the corresponding base exit and then wake the creature.

**Invariants** — waking happens on **both** paths, and that symmetry is the file's real contract. A
leaf that is pre-empted — by a hit, by a sighting, by the pack composite changing its mind — must
still wake the creature, or it is left permanently deaf and blind while standing up and walking
around. This is the clearest case in the directory of why the state machinery distinguishes a clean
exit from a forced one at all, and of why both must be implemented even when their bodies are
identical.

The leaf has no completion test, so it never ends on its own: the composite above decides when
sleeping is over.
