# src/xrGame/alife_human_brain_save.h

> A disabled file kept on purpose: the offline ammunition model, the meet-or-fight decision, and how a squad divided loot. Not compiled; the header comment forbids deleting it.

**Needs** — _(none)_
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T4: nothing compiles.

## Purpose

Three routines from the off-screen combat and trading model, commented out and preserved.
They are the only surviving statement of how abstract combat consumed ammunition and how a
squad shared what it found. A rebuilder reviving off-screen combat starts here; a rebuilder
reproducing the shipped games ignores it.

## The offline ammunition model

**Contract** — One abstract attack, answering whether the attacker can attack again.

```text
FUNCTION perform_attack() -> bool
  IF no weapon is selected THEN RETURN false
  SWITCH the weapon's inventory slot
    CASE the natural-weapon slot
      RETURN true                    # claws and teeth never run out
    CASE the throwable slot
      find the weapon among this character's children      # FAIL if absent
      release it                                           # a grenade is consumed whole
      RETURN false                                         # and cannot be thrown again
    OTHERWISE
      REQUIRE the weapon has ammunition
      IF this character does not have unlimited ammunition
        spend one round
      IF rounds remain THEN RETURN true
      # Out of ammunition: destroy every ammunition box in the inventory that this
      # weapon could have drawn from, and deselect the weapon.
      release every partially-used ammunition item matching the weapon's ammunition
      deselect the weapon
      RETURN false
```

**Invariants** — Ammunition is modelled at two levels and they are reconciled here: a
*round count* on the weapon, and *boxes* in the inventory. The weapon's count is what is
spent; the boxes are destroyed wholesale when it runs out, because offline combat does not
model reloading and a box left behind would let the same weapon fire again after the
character came online. The unlimited-ammunition flag is a per-character trader property,
which is how story characters never run dry.

The three slots — natural weapon, throwable, everything else — are the whole taxonomy
offline combat needs. A rebuild reviving this model needs the same three cases and can
ignore the rest of the weapon system.

## The meet-or-fight decision

**Contract** — On meeting another creature, and only when both parties are creatures:
a *friend* is interacted with; anyone else is attacked if the detection was mutual or if the
odds estimate says attack, and ignored otherwise. Any other pairing — a creature meeting an
anomaly — is always an attack.

**Invariants** — Mutual detection forces a fight regardless of the odds. Two enemies who have
each seen the other cannot both walk away; only a one-sided sighting allows the weaker party
to decline.

## Squad loot division

**Contract** — A squad dividing spoils makes **two passes** over its members: in the first
each member takes only its minimum needs, and in the second each takes what it wants from
what is left.

**Invariants** — The two-pass shape is the interesting part and is worth keeping: a single
greedy pass lets the first member strip the pile and leaves the last with nothing. Two
passes guarantee everyone is equipped before anyone is well equipped.
