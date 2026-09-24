# src/xrGame/ai/monsters/state_defs.h

> The identifier space every creature state is registered and selected by: a bit-field encoding where the family lives in a high bit and the member in the low bits, so that "is this any kind of attack" is one masked comparison.

**Needs** — _(none beyond core string handling)_
**Used by** — [`psy_dog_state_psy_attack_inline.h`](pseudodog/psy_dog_state_psy_attack_inline.h.md) · [`state.cpp`](state.cpp.md) · [`state.h`](state.h.md) · [`state_inline.h`](state_inline.h.md) · [`state_manager.h`](state_manager.h.md)
**Tier floor** — T3: a constant table and one masked comparison

## Purpose

State identifiers are not opaque tags. They are structured so that a *family* of states —
rest, eat, attack, panic, reaction-to-hit, reaction-to-sound, controlled, threaten,
find-enemy, squad, custom — can be tested for with a single mask, which is what lets code
outside the state tree ask "is this creature attacking anything at all" without enumerating
every attack variant. The encoding is the decision on this page; the list of names is data.

## The encoding

```text
RECORD StateId : int (32-bit, bit-field)
  bit 15                : the marker bit, set on every real identifier
  bits 16..30           : the family; exactly one set per family
  bits 0..14 (low)      : the member within the family, a small counting number
```

The families occupy successive bits above the marker: rest at bit 16, eat at 17, attack at 18,
panic at 19, reaction-to-hit at 20, dangerous-sound at 21, interesting-sound at 22, controlled
at 23, threaten at 24, find-enemy at 25, squad at 26, custom at 30. A member is the family
value with a small number bit-or'd into the low bits. "Unknown" is all bits set.

The family test is: *the identifier contains every bit of the family value, and is not
"unknown"*. That is a containment test, not an equality test, and three consequences follow
from it — the first intended, the other two not.

**Intended: sub-families nest.** Some identifiers are deliberately built from two family bits
at once. A vampire attack or a burer attack carries both the custom bit and the attack bit, so
those states answer *yes* to "is this an attack" while still being distinguishable as custom
ones. The predator states the bloodsucker uses are built the same way. This is the reason the
test is containment.

**Unintended: the low bits collide.** Members are counted, not bit-allocated, so their bits
overlap. The attack family numbers its members 1 through 25, and members 16 through 19 were
declared as a sub-family ("camp" and its three phases). Because 16 is a power of two, *every*
attack member from 16 upward has that bit set — so the five members numbered 20 to 25,
including the psi attack, the home-point moves and the attack-on-run, all answer *yes* to "is
this the camp state". Nothing in the chapter currently asks that question, which is why it has
never surfaced, and a rebuild should allocate sub-family bits rather than reuse counting
numbers.

**Unintended: one family is nested inside another by accident of numbering.** The
"heard a call for help" states are numbered as members 3, 4 and 5 of the *interesting sound*
family rather than given a family of their own. So a creature reacting to a call for help
answers yes to "is this creature investigating an interesting sound". Several selectors in the
chapter test the two conditions in sequence and depend on ordering rather than on the mask to
tell them apart.

## What is authored and what is not

Every identifier on this page is compiled in. Nothing here comes from configuration: the
*set* of behaviours a creature can have is code, and only the numbers that tune them are data.
That split runs through the whole chapter — a modder can make a dog faster, more cowardly or
blinder, but cannot give it a state the engine does not already compile.

Several identifiers exist here with no live registration anywhere. Those are named in the
files that were supposed to register them; the notable ones are the burer's scanning state
(registered, but the ability that drives it is never instantiated — see
[`scanning_ability.h`](scanning_ability.h.md)) and the chimera's threaten tree (registered only
from a commented-out line).

## `is_state`

**Contract** — answers whether an identifier belongs to a family.

```text
FUNCTION is_state(id, family) -> bool
  RETURN (id CONTAINS ALL BITS OF family) AND (id IS NOT unknown)
```

The explicit exclusion of "unknown" is necessary because "unknown" is all-bits-set and would
otherwise belong to every family at once.

## `make_xrstr`

**Contract** — maps an identifier to a human-readable name, for the debug overlay and the log.
Pure lookup; unmapped values yield a placeholder. Implemented in
[`state.cpp`](state.cpp.md).
