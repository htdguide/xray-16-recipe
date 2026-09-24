# src/xrGame/ai/stalker/ai_stalker_misc.cpp

> What a stalker considers worth picking up, worth fighting, and worth shouting about — plus the squad telepathy that lets one stalker inherit another's enemy.

**Needs** — [`ai_stalker.h`](ai_stalker.h.md) · [`ai_stalker_impl.h`](ai_stalker_impl.h.md) · [`ai_stalker_space.h`](ai_stalker_space.h.md) · [`memory_manager.h`](../../memory_manager.h.md) · [`item_manager.h`](../../item_manager.h.md) · [`enemy_manager.h`](../../enemy_manager.h.md) · [`danger_manager.h`](../../danger_manager.h.md) · [`visual_memory_manager.h`](../../visual_memory_manager.h.md) · [`agent_manager.h`](../../agent_manager.h.md) · [`agent_member_manager.h`](../../agent_member_manager.h.md) · [`agent_explosive_manager.h`](../../agent_explosive_manager.h.md) · [`agent_location_manager.h`](../../agent_location_manager.h.md) · [`relation_registry.h`](../../relation_registry.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: filter predicates and one cross-entity information transfer

## Purpose

Four unrelated policies, collected because each is one short answer the shared perception
machinery asks the stalker.

The one a rebuilder must not miss is **`process_enemies`** — the mechanism by which a squad
behaves as a unit. It is the difference between four stalkers who happen to be near each
other and a squad that turns to face an ambush together.

## Constants

```text
TOLLS_INTERVAL             =  2 seconds     # delay before mourning a fallen squadmate
GRENADE_INTERVAL           =  0             # dead: see Notes
FRIENDLY_GRENADE_ALARM_DIST = 5 metres      # how close a friendly grenade must be to warn about
DANGER_INFINITE_INTERVAL   = ~16.7 hours    # effectively permanent
DANGER_EXPLOSIVE_DISTANCE  = 10 metres      # radius of the danger a loose explosive stamps
```

## `useful(item)` — what is worth picking up

**Contract** — the item-perception filter, called for every item the stalker becomes aware
of. It answers "is this item worth remembering", and — because it is the natural choke point
— it also *raises the alarm about explosives* as a side effect.

```text
FUNCTION useful(item) -> bool
  IF the item is an EXPLOSIVE and also an inventory item
    stamp a permanent 10-metre danger location on it for the whole squad
        # a loose grenade or mine on the ground is a place nobody should walk

  IF the item is an EXPLOSIVE with a live parent (somebody threw it)
    register it with the squad's explosive manager
    IF the thrower is a living entity
      record a GRENADE danger, perceived visually, attributed to the thrower
        # this is how a stalker knows to run from a thrown grenade, and who threw it

  IF the shared item memory does not already consider it useful  THEN RETURN false
  IF it is not an inventory item, or not flagged useful to NPCs  THEN RETURN false
  IF it is a BOLT                                                THEN RETURN false
  IF my inventory cannot take it                                 THEN RETURN false
  RETURN true
```

**Invariants** — a filter with a side effect is a design smell and a rebuild may want to
split it, but the *coupling is load-bearing*: explosives must be noticed at the moment they
enter perception, and this is the only function called at that moment. Splitting it means
finding another hook with the same timing.

**Notes** — bolts are excluded explicitly. The player's throwable bolt is an inventory item
flagged useful, and without this line stalkers would spend firefights collecting bolts.

## `evaluate(item)` — which item first

**Contract** — returns the squared distance to the item, floored at a tiny positive value.
The item manager sorts ascending, so the stalker fetches the nearest item first. There is no
value weighting at this stage; value is decided later, by the weapon-choice search in
[`ai_stalker_fire.cpp`](ai_stalker_fire.cpp.md).

## `useful(enemy)` — what is worth fighting

**Contract** — two gates. The squad's shared enemy manager must consider the candidate a
useful enemy *for this stalker* — which is how a squad distributes targets rather than all
four shooting the same man — and the stalker's own enemy memory must agree.

## `tfGetRelationType`

**Contract** — reputation first, base entity rule second. The reputation registry is
consulted only when the other party is an inventory owner and is not a creature; creatures
have no reputation. When the registry has no opinion, the base rule — faction and team —
decides.

## `react_on_grenades`

**Contract** — the squad-level grenade reaction. Runs when the squad's member record has an
unprocessed grenade event. Only stalkers with group behaviour react vocally.

```text
FUNCTION react_on_grenades()
  reaction = my squad member record's grenade reaction
  IF nothing to process THEN RETURN
  IF the delay has not elapsed THEN RETURN          # the delay is zero; see Notes

  IF the grenade is a missile AND my squad uses group behaviour
    thrower = whoever the grenade's parent identifies
    IF the thrower is an enemy
      say "grenade alarm"
    ELSE IF the grenade landed within 5 metres of me
      # a FRIENDLY grenade this close is a warning to my own side, and the
      # vocalisation is timed to the grenade's own fuse: it is scheduled to
      # start before the explosion, not after
      remaining = grenade's destruction time - now(), floored at zero
      say "friendly grenade alarm", window (remaining + 1.0 s, remaining + 1.5 s)

  clear the reaction
```

**Invariants** — the friendly-grenade warning is the only vocalisation in the game whose
timing is derived from another object's lifetime. A rebuild must keep the fuse-relative
scheduling or the warning arrives after the blast.

**Notes** — the delay constant is written as zero times a thousand, so the delay test always
passes. The whole delay mechanism is inert. A disabled block beside it would have stamped a
danger location around the grenade with a radius and a duration derived from its fuse; that
job now falls to the item filter above, which stamps a fixed ten metres permanently instead.

## `react_on_member_death`

**Contract** — when a squadmate goes down and two seconds have passed, a stalker with group
behaviour says one of two things: *tolls* if the squadmate is dead, *wounded* if it is only
down. Both are scheduled in a window between two and three seconds out, so that several
squad members mourning at once do not speak over each other exactly.

## `process_enemies` — the squad telepathy

**Contract** — the mechanism that makes a squad react together. Runs on every scheduled
update, and returns immediately if the stalker already has an enemy.

```text
FUNCTION process_enemies()
  IF I already have a selected enemy THEN RETURN

  FOR EACH object in my visual memory
    IF I cannot currently see it under my squad mask THEN CONTINUE
    IF it is not a stalker                            THEN CONTINUE
    IF it is my enemy                                 THEN CONTINUE
    IF it is not alive                                THEN CONTINUE

    IF that stalker has no enemy either
      # he has no enemy but he may have noticed DANGER; if I have none, take his
      IF I have no selected danger AND he has one
        copy his selected danger into my danger memory
      CONTINUE

    IF I cannot see him at this very moment            THEN CONTINUE

    # take his enemy as something I have "seen somewhere" - not as a confirmed
    # sighting. The enemy enters my memory as an object to look for, and my own
    # perception must still confirm it before I will shoot.
    make his enemy visible-somewhen to me
    BREAK
```

**Invariants**

- The transfer is *one hop*: I take an enemy from someone I can see right now, and they took
  theirs from someone they could see. Alarm therefore propagates through a squad at one hop
  per scheduled update rather than instantly, which is what makes a squad turn raggedly
  rather than in unison.
- The enemy is transferred as *remembered*, not as *seen*. The receiving stalker gets a
  belief it must confirm, so it turns and searches rather than shooting immediately. That
  distinction is the whole reason the mechanism does not feel like cheating.
- Danger is transferred under the opposite condition — only from a friend who has *no*
  enemy. A friend who is fighting transmits the enemy; one who is merely uneasy transmits
  the unease.

**Notes** — the "I must see him at this very moment" gate is an explicit later correction,
marked as such in the source. Without it a stalker could inherit an enemy from a squadmate
it merely *remembered* seeing, which let alarm spread through walls. A rebuild must include
it; the behaviour without it is visibly wrong.
