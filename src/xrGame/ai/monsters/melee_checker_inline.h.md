# src/xrGame/ai/monsters/melee_checker_inline.h

> The authored numbers of the melee window, the per-bout reset, and the arithmetic that turns the adapting inner edge into a matching outer edge.

**Needs** — [`melee_checker.h`](melee_checker.h.md) · [Seam: Script virtual machine](../../../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — [`melee_checker.cpp`](melee_checker.cpp.md) · [`melee_checker.h`](melee_checker.h.md)
**Tier floor** — T3: reading four numbers and doing arithmetic on them

## Purpose

Holds the parts of the melee judge that are short enough to be worth keeping at every call
site. In a rebuild there is no reason for the split — merge these into
[`melee_checker.cpp`](melee_checker.cpp.md). What survives is the set of authored keys and the
relationship between the two edges of the window.

## `load`

**Contract** — reads four required numbers from a creature's configuration section. No
defaults: a creature whose section omits any of them fails to load.

| Key | Meaning |
|---|---|
| `MinAttackDist` | The inner edge at full confidence — the furthest out the creature will ever commit to a swing |
| `MaxAttackDist` | The outer edge at full confidence — the range past which a bout is abandoned |
| `as_min_dist` | The floor the inner edge may be driven down to by repeated misses |
| `as_step` | How far one confirmed run of agreeing swings moves the inner edge |

**Notes** — the abbreviated keys (`as_` for the adaptive pair) are part of the shipped
configuration data and are frozen by it; they cannot be renamed in a rebuild that loads the
original game's files.

Nothing checks that `as_min_dist` is below `MinAttackDist` or that `MinAttackDist` is below
`MaxAttackDist`. Data that violates either produces a window that is empty or inverted, and
the creature then either never swings or never disengages.

## `begin_attack`

**Contract** — seeds the swing history as *all connected* and puts the inner edge at its
widest. Called when the creature enters its attack state.

```text
FUNCTION begin_attack()
  fill swing_history with true          # start optimistic
  current_min = min_attack_distance     # start at the widest edge
```

**Notes** — starting optimistic means the very first miss of a bout cannot move the edge (it
breaks the run rather than confirming one); it takes two misses. A creature therefore always
opens a bout at its nominal reach and only tightens if the opening swings genuinely fail.

## `min_distance` / `max_distance`

**Contract** — the live edges of the melee window.

```text
FUNCTION min_distance() -> real
  RETURN current_min

FUNCTION max_distance() -> real
  RETURN max_attack_distance - (min_attack_distance - current_min)
```

**Notes** — the outer edge is the authored outer edge shifted by exactly the amount the inner
edge has been pulled in. So the window's *width* is the authored
`MaxAttackDist - MinAttackDist` under every adaptation, and only its placement moves. That
invariant is the whole point of the second line and is stated nowhere in the original: a
rebuild that clamps the outer edge independently will change the hysteresis, and with it how
often creatures stutter at the edge of contact.
