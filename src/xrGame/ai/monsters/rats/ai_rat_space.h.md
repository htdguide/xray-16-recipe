# src/xrGame/ai/monsters/rats/ai_rat_space.h

> The rat's five sounds and the masks that decide which of them may interrupt which.

**Needs** — _(none)_
**Used by** — [`ai_rat.cpp`](ai_rat.cpp.md) · [`ai_rat_fire.cpp`](ai_rat_fire.cpp.md) · [`rat_state_activation.cpp`](rat_state_activation.cpp.md)
**Tier floor** — T3: two constant tables

## Purpose

The sound player the rat uses selects among registered sounds by a *mask* rather than by a
priority number, so that "play this only if nothing incompatible is sounding" is one bitwise
test. This file declares the rat's five sound identifiers and the mask each is registered with.

## The identifiers

```text
ENUM RatSound
  die, injuring, attack, voice, eat
```

Registered in [`ai_rat.cpp`](ai_rat.cpp.md) against the head bone, each with a clip name read
from the creature's configuration section.

## The masks

```text
ENUM RatSoundMask
  any_sound  = 0                          # matches nothing; the "no constraint" value
  die        = every bit set
  injuring   = every bit set
  voice      = highest bit, plus bit 0
  attack     = second-highest bit, plus bit 1
  eat        = second-highest bit, plus bit 2
```

**Invariants** — the encoding has two halves and only the high half is load-bearing. Death and
injury carry **every** bit, so they are compatible with — and therefore may interrupt or
coexist with — anything. Attack and eating share one high bit, so they are mutually
interchangeable. Voice sits alone on the *other* high bit, so idle chittering is in a class of
its own and does not contend with feeding or biting.

The low bits (zero, one, two) are distinct per sound and give each one an identity within its
class, so that two rats' idle sounds are not treated as the same sound event.

**Notes** — why voice was given the top bit and attack and eating the one below it is not
recoverable. The effect is that a rat can chitter while eating, which is audible in game, and
that a rat cannot chitter while biting. Whether that pairing was chosen or fell out of the bit
assignment, nothing says.

A sixth identifier and a sixth mask are declared as terminators, both all-bits-set, and neither
is used. The "any sound" mask of zero is also never passed anywhere by the rat.
