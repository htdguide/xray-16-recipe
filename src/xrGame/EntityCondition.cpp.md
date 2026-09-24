# src/xrGame/EntityCondition.cpp

> Everything a living creature can be in the middle of: health, stamina, radiation, psychic health, morale and a set of open wounds, all advanced against in-world time and all changed through one accumulator per frame.

**Needs** — [`EntityCondition.h`](EntityCondition.h.md) · [`entity_alive.h`](entity_alive.h.md) · [`Wound.h`](Wound.h.md) · [`InventoryOwner.h`](InventoryOwner.h.md) · [`Inventory.h`](Inventory.h.md) · [`CustomOutfit.h`](CustomOutfit.h.md) · [`ActorHelmet.h`](ActorHelmet.h.md) · [`hit_immunity.h`](hit_immunity.h.md) · [`Level.h`](Level.h.md) · [`game_cl_base.h`](game_cl_base.h.md) · [`Common/object_broker.h`](../Common/object_broker.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md)
**Used by** — reached through its declarations in [`EntityCondition.h`](EntityCondition.h.md); callers name that, not this file.
**Tier floor** — T2: scalar integration against a clock, plus a save format

## Purpose

This is the game's model of *being alive and damaged*. Every creature, the player included,
owns one of these; the player's subclass adds hunger, thirst and drunkenness on top.

Two ideas hold the file together.

**Nothing writes a condition value directly.** Every change — a hit, a bandage, a second of
radiation, a wound bleeding — is added into a *delta* accumulator for that value. Once per
update, the deltas are consumed, the values are clamped, and the accumulators are zeroed.
This is why several simultaneous sources of damage cannot race, why death is decided in one
place, and why a rebuild must keep the accumulate-then-apply split even though it looks like
indirection.

**A hit is not damage.** It is a quantity with a *type*, and the type decides which value it
lands on, which of the twelve resistance numbers apply, and whether it opens a wound. The
switch over hit types is the most load-bearing block in the file, and it is a table, not an
algorithm.

The values are all normalized to zero-to-one, except health, which is allowed to go very
slightly negative so that "dead" and "exactly zero" are distinguishable.

## State

```text
RECORD ConditionValues              # all in [0,1] except health, floored at -0.01
  health, max_health        : real
  power, power_max          : real  # stamina; max_power is clamped to [0.1, 1]
  radiation, radiation_max  : real
  psy_health, psy_max       : real
  morale, morale_max        : real

RECORD ConditionDeltas              # accumulated during a frame, consumed by UpdateCondition
  delta_health, delta_power, delta_radiation, delta_psy_health : real
  delta_morale, delta_circumspection                           : real

RECORD ConditionRates               # read from configuration, all per second
  radiation_v          : real   # how fast an absorbed dose decays
  radiation_health_v   : real   # how much health a unit of dose costs per second
  psy_health_v         : real   # psychic health regeneration
  morale_v             : real
  bleeding_v           : real   # how much health a unit of open wound costs
  wound_incarnation_v  : real   # how fast wounds close on their own
  health_restore_v     : real   # passive regeneration; defaults to none
  circumspection_v     : real   # fixed at 0.01 and never read from data

RECORD HitSplit
  health_hit_part      : real   # fraction of a hit that becomes health loss
  power_hit_part       : real   # fraction of a hit that becomes stamina loss
  hit_bone_scale       : real   # per-bone multipliers, pushed in by the damage manager
  wound_bone_scale     : real   #   just before a hit is processed

RECORD LastChance                   # the "two hits to kill" rule
  kill_hit_threshold   : real   # above this health, a lethal hit is survivable
  last_chance_health   : real   # what you are left with instead
  invulnerable_time    : real   # for this long afterwards, nothing can kill you
  invulnerable_until   : real

RECORD Wounds
  wounds               : list<wound>   # at most one per bone; bone index < 64
  min_wound_size       : real          # below this a wound is considered closed
  is_bleeding          : bool

RECORD Boosts                        # twelve temporary modifiers, all additive
  burn, shock, radiation, telepathic, chemical burn, explosion, strike,
  fire wound, wound            : immunity bonuses   # SUBTRACTED from the immunity multiplier
  radiation, telepathic, chemical burn : protection # SUBTRACTED from incoming power

  last_hit_by          : object, and its identifier
  can_be_harmed        : bool     # and only on the authoritative side
  delta_time           : real     # seconds of in-world time since the last update
  time_valid           : bool
```

Invariants:

- **at most one wound per bone.** A second hit on the same bone deepens the existing wound
  rather than adding another. The bone index must be below 64, which is a limit inherited
  from the packed per-bone masks used elsewhere.
- the delta accumulators are zero at the end of every `UpdateCondition` and nowhere else.
- `can_be_harmed` is the conjunction of a settable flag **and** being on the authoritative
  side. A client never reduces its own health; it is told.
- healing is always permitted even when harming is not: a positive health change bypasses
  the harm check entirely.

## `UpdateCondition` — the one place values change

**Contract** — consume one update's worth of deltas. Called once per scheduled update of the
owning creature. Does nothing at all once health has reached zero.

```text
FUNCTION update_condition()
  IF health <= 0 THEN RETURN

  # a chain of four death-cause tests, each asked before the step that could cause it
  IF the accumulated delta already kills THEN
    critical = true; tell the object it died of a HIT
  ELSE IF the delta is negative THEN
    tell the object how much health a hit has left it with

  UpdateHealth()              # bleeding, passive regeneration, wound closing
  IF not yet critical AND the delta now kills THEN
    critical = true; tell the object it died of WOUNDS

  UpdatePower(); UpdateRadiation()
  IF not yet critical AND the delta now kills THEN
    critical = true; tell the object it died of RADIATION

  UpdatePsyHealth(); UpdateEntityMorale()

  # the last-chance rule
  IF the invulnerability window has expired THEN
    IF health is above the kill threshold AND the delta would take it below zero THEN
      health = last_chance_health
      open an invulnerability window
    ELSE
      health = health + delta_health

  add every other delta to its value
  zero every delta
  clamp health to [-0.01, max_health] and every other value to [0, its maximum]
```

**Invariants** — the death-cause chain is the reason this function is not simply "add the
deltas". The game needs to know *what killed you* — a bullet, blood loss or radiation — for
the death animation, the kill message and the statistics, and the only way to know is to ask
after each contributing step. The order of the three tests is the order of the causes'
priority.

**Notes** — the **last-chance rule** is the mechanic that makes an enemy take two hits to
kill you rather than one: if you were healthy enough (above the kill threshold) and a single
hit would be fatal, you are left at a sliver of health and made invulnerable for a fraction
of a second. Both numbers and the window are per section and default to zero, which disables
the rule. It is what makes the game's difficulty levels feel different without changing any
weapon's damage.

The invulnerability window is timed against **real** time while everything else in this file
is timed against **in-world** time. That is not obviously deliberate: it means the window's
length is unaffected by the game's time compression, which is probably right for a
reflex-scale mechanic and is inconsistent with everything around it.

## `ConditionHit` — the hit-type table

**Contract** — apply one hit. Returns the wound it opened, if any. This is where a hit
becomes a change in condition.

```text
FUNCTION condition_hit(hit) -> optional<wound>
  remember who struck
  power = hit.damage
  power = HitOutfitEffect(power, type, bone, armour piercing, may_wound)
          # armour and helmet get first refusal, and may veto the wound

  SWITCH on hit.type
    telepathic     : power = power - telepathic protection, floored at zero
                     power = power * (immunity(type) - telepathic boost)
                     psychic health takes it; health and stamina take their shares
                     no wound
    burn, light burn : power = power * (burn immunity - burn boost)
                     health share is ALSO scaled by the bone's hit scale
                     no wound
    chemical burn  : protection subtracted, then immunity; no wound
    shock          : immunity only; no wound
    radiation      : protection subtracted, then immunity
                     the whole hit becomes absorbed DOSE, not damage; RETURN none
    explosion      : immunity only; MAY wound
    strike, physical strike : immunity only; no wound
    fire wound     : immunity, and the bone's hit scale; MAY wound
    wound          : immunity, and the bone's hit scale; MAY wound
    otherwise      : fatal — an unknown hit type is a programming error

  IF a wound is permitted AND health is above zero THEN
    RETURN AddWound(power * the bone's wound scale, type, bone)
  RETURN none
```

**Invariants** — three rules run through the whole table and are worth stating once:

- a **protection** boost is subtracted from the incoming power (and the result floored at
  zero), so enough protection makes a hit of that type harmless outright. Only the three
  environmental types have one.
- an **immunity** boost is subtracted from the immunity *multiplier*, so it scales the hit
  down proportionally. Every type has one. The sign is counter-intuitive: a positive boost
  makes the multiplier smaller, which is less damage.
- only four types can open a wound — explosion, fire wound, wound, and only then if the
  armour did not veto it. Burns, shocks, chemical burns, telepathy and strikes never bleed.

Radiation is the one type that never touches health directly. It adds to a dose that then
costs health continuously, which is why radiation kills slowly and cannot be out-healed
without also clearing the dose.

**Notes** — the "special hit to self" case (the object hits itself with no bone named) is
computed and then used only to suppress a debug message. A commented-out line shows it once
decided whether a burn could wound yourself. Not recoverable as an intention.

## `HitOutfitEffect` and `HitPowerEffect`

**Contract** — give the worn armour and helmet a chance to reduce a hit and to veto the
wound it would open. Applies the outfit first and the helmet to the outfit's result.

```text
FUNCTION hit_outfit_effect(power, type, bone, armour_piercing, may_wound) -> real
  IF the object carries no inventory THEN RETURN power unchanged
  outfit = whatever is in the outfit slot; helmet = whatever is in the helmet slot
  IF neither THEN RETURN power unchanged
  IF outfit THEN power = outfit.through_armour(power, bone, ap, may_wound, type)
  IF helmet THEN power = helmet.through_armour(power, bone, ap, may_wound, type)
  RETURN power
```

**Invariants** — the order matters and the composition is sequential, not parallel: a helmet
reduces what the outfit already let through. Both are given the *bone*, which is how a
helmet protects only head hits.

`HitPowerEffect` scales stamina loss by the outfit's own factor, and — with no outfit at all
— **halves it**. A naked creature loses less stamina than one wearing the flimsiest armour
whose factor exceeds one half. That is almost certainly not intended and it is the shipped
behaviour.

## `UpdateHealth`, `UpdateRadiation`, `UpdatePsyHealth`, `UpdateEntityMorale`, `UpdatePower`

**Contract** — the four continuous processes, each contributing to a delta rather than to a
value.

```text
FUNCTION update_health()
  loss = BleedingSpeed() * delta_time * bleeding rate
  is_bleeding = loss is non-zero
  delta_health = delta_health - loss (only if harmable)
  delta_health = delta_health + delta_time * health restore rate
  close every wound by (wound incarnation rate * delta_time)

FUNCTION update_radiation()
  IF the dose is positive THEN
    delta_radiation = delta_radiation - decay rate * delta_time
    delta_health    = delta_health - (radiation health rate * dose * delta_time)

FUNCTION update_psy_health()   delta_psy_health += regeneration rate * delta_time
FUNCTION update_morale()       IF below maximum, delta_morale += rate * delta_time
FUNCTION update_power()        nothing — the base creature has no stamina model
```

**Notes** — stamina regeneration lives in the player's subclass, not here, because only the
player spends stamina. The base is empty rather than absent so the subclass has somewhere to
hook.

Radiation damage is proportional to the *current dose*, so a large dose kills quickly and a
small one is survivable — and the dose decays at a fixed rate regardless of its size, which
makes recovery linear while the damage is not.

## Wounds

### `AddWound`, `ChangeBleeding`, `UpdateWounds`, `BleedingSpeed`, `ClearWounds`

**Contract** — a wound is per bone, deepens when hit again, closes over time, and costs
health while open.

```text
FUNCTION add_wound(power, type, bone) -> wound
  REQUIRE bone < 64 or no bone
  find the existing wound on that bone, or make one
  add a hit of (power * a random factor between 0.5 and 1.5) of that type
  RETURN it

FUNCTION bleeding_speed() -> real
  RETURN the MEAN of every wound's size, or zero with no wounds
```

**Invariants** — bleeding is the **mean** wound size, not the sum. So two small wounds bleed
at the average of the two, and a creature with many shallow wounds bleeds no faster than one
with a single wound of that depth. Whether that is a decision or a bug is not recoverable —
it is a design a rebuild must reproduce either way, because every bandage and medkit in the
shipped data is tuned against it.

**Notes** — the random factor of 0.5 to 1.5 on each hit's contribution is why two identical
shots produce different wounds. It is the only stochastic element in the condition model.

`ChangeBleeding` is used for two opposite purposes: the continuous closing of wounds over
time, and a bandage's instant heal. Both are "reduce every wound by this fraction"; a wound
that falls below the minimum size marks itself for destruction and is swept out by
`UpdateWounds`.

## `UpdateConditionTime`

**Contract** — measure in-world time since the last update and publish it as the delta every
rate above is multiplied by. The very first call after a load or a spawn yields **zero** and
discards whatever deltas had accumulated.

```text
FUNCTION update_condition_time()
  now = in-world game time in single player, server time otherwise
  IF the previous reading is valid THEN
    delta_time = (now - previous) / 1000, or zero if the clock went backwards
  ELSE
    delta_time = 0; mark valid; zero every accumulator
  previous = now
```

**Invariants** — the zero-first-tick rule is what stops a creature that has been offline for
in-world days from bleeding to death in one frame the moment it comes online. The clock this
reads is the one the alife simulation advances, which may have jumped hours.

The two clock sources are not interchangeable: in single player the in-world clock is what
the player experiences (and it is compressed relative to real time), in a networked session
the server's clock is the only shared one.

## `IsLimping`

**Contract** — a creature limps when the *product* of its stamina and its health falls below
a threshold. Disabled unless the section opts in.

**Notes** — the product, not either value, is the decision: a healthy exhausted creature and
a wounded fresh one both limp, and a creature that is both is well past the threshold. The
default threshold of one half means limping starts around 70 percent of each.

## `ApplyInfluence`, `ApplyBooster`

**Contract** — consume a consumable. An influence is an immediate bundle of changes (health,
stamina, satiety, dose, wound healing, maximum stamina, alcohol) applied at once. A booster
is a timed modifier; the base implementation accepts it and does nothing.

**Notes** — the base does nothing with boosters because only the player has the map of active
boosts and the timer that expires them. The seventeen boost kinds are named in
[`EntityCondition.h`](EntityCondition.h.md) and each maps to exactly one configuration key.

## `save`, `load`

**Contract** — the condition's slice of the save. One flag says whether the creature was
alive; a dead creature saves nothing else.

```text
alive : bool
IF alive
  power, radiation, morale, psy_health : real
  wound count : int (8-bit)
  each wound's own serialization
```

**Invariants** — **health is not saved here.** It lives on the entity itself, which is why
the flag is needed at all. And because the wound count is a byte, a creature cannot carry
more than 255 wounds — which the one-wound-per-bone rule already guarantees, since bones are
capped at 64.

The load marks the clock invalid, so the first update after restoring a save contributes no
elapsed time. Without that, everything that ticks would be advanced by the entire gap
between the save and the load.

## `remove_links`

**Contract** — when the object recorded as the last attacker is destroyed, **the creature
becomes its own attacker** rather than having none.

**Notes** — the substitution rather than a clear is deliberate: downstream code reads the
attacker without checking, and pointing it at the victim keeps every such read valid. A
creature that dies of wounds inflicted by someone who has since been deleted is recorded as
having killed itself, which is the least wrong available answer.

## `SConditionChangeV::load`, `SMedicineInfluenceValues::Load`, `SBooster::Load`

**Contract** — the three configuration readers. The rate block reads its seven keys with an
optional suffix, so that one section can carry several rate sets — that is how difficulty
levels are expressed in the shipped data. The circumspection rate is assigned a compiled-in
0.01 and never read from data.

`SBooster::Load` is a seventeen-way switch from boost kind to configuration key; the key
names are also listed as an array in the header so the user interface can describe a
consumable without instantiating one. The two lists must stay in step, and nothing enforces
it.
