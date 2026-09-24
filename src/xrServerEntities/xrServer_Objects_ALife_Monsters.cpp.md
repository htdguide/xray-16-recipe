# src/xrServerEntities/xrServer_Objects_ALife_Monsters.cpp

> Every record that is alive or is a zone: its fields, its three serializations, the version gates that let a 2007 save still load, and the algorithm that turns "profile *X*" into a named individual with a face, a name and a wallet.

**Needs** — [`xrServer_Objects_ALife_Monsters.h`](xrServer_Objects_ALife_Monsters.h.md) · [`xrServer_Objects_ALife_Items.h`](xrServer_Objects_ALife_Items.h.md) · [`alife_space.h`](alife_space.h.md) · [`character_info.h`](character_info.h.md) · [`specific_character.h`](specific_character.h.md) · [`alife_human_brain.h`](alife_human_brain.h.md) · [`alife_monster_brain.h`](alife_monster_brain.h.md) · [`Common/object_broker.h`](../Common/object_broker.h.md) · [Data: save format](../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — reached through its declarations in [`xrServer_Objects_ALife_Monsters.h`](xrServer_Objects_ALife_Monsters.h.md); callers name that, not this file.
**Tier floor** — T1: on-disk and on-wire layouts, with version-gated field presence.

## Purpose

The largest single file in the chapter, and the one conformance criterion 7 leans on
hardest. Almost every entity in a shipped level that is not an item is one of these records,
and a save game from the original engine is mostly this file's output.

Read it as three things laid on top of each other: a **field list** per record; a **version
changelog** written as read-time gates; and one **algorithm** — profile resolution — that
does not serialize anything at all but decides what a stalker looks like, what faction they
fight for and how much money is in their pocket.

## The terrain-preference parser

Before any record: a shared helper that turns an authored terrain preference into a list of
masks over the navigation graph's location types.

**Contract** — reads either a **configuration section**, in which each line whose value has
exactly one entry per location type is one mask; or a **single line** holding a whole number
of masks back to back. Answers a list of masks. Never fails.

```text
FUNCTION parse_terrain(text) -> list<mask>
  IF text names a configuration section with at least one line
    FOR EACH line IN that section
      IF the line's value has exactly LOCATION_TYPE_COUNT entries
        APPEND those entries AS a mask
      ELSE
        SKIP the line          # silently: a malformed line is not an error
  ELSE
    n = (entry count of text) rounded DOWN to a multiple of LOCATION_TYPE_COUNT
    FOR EACH group of LOCATION_TYPE_COUNT entries in the first n
      APPEND the group AS a mask
  IF no mask was produced
    APPEND the all-255 mask   # "any terrain"
```

**Invariants** — **the fallback is "accept everything"**, expressed as every byte set. A
creature whose terrain preference failed to parse can go anywhere, which fails loud in
gameplay rather than at load and is why nobody notices a typo in a terrain line. The
rounding-down in the single-line form means a trailing partial mask is discarded, also
silently.

## `CSE_ALifeTraderAbstract` — the identity mixin

The mixin that gives a record a wallet and a name. Mixed into the trader, the actor and
every human.

```text
RECORD TraderMixin
  money              : int (32-bit)
  specific_character : text      # the resolved individual's identifier
  trader_flags       : int (32-bit, bit field)   # bit 0: infinite ammunition
  character_profile  : text      # the authored template; defaults to "default"
  community_index    : int (32-bit, signed)      # -1 means unset
  rank               : int (32-bit, signed)      # -1 means unset
  reputation         : int (32-bit, signed)      # -1 means unset
  character_name     : text      # the display name, already localized
  corpse_lootable    : bool      # written as one byte
  corpse_closed      : bool      # written as one byte
```

Held but not serialized: the maximum carried mass (from the section), the portrait name, and
two scratch lists used during profile resolution.

### Save write

Money, the resolved individual, the flag word, the profile, the faction index, the rank, the
reputation, the display name, then the two corpse bytes.

**Invariants** — **the tools build writes placeholders where the game build writes resolved
values**: an empty individual and three "unset" markers. The field count and the widths are
identical, so a spawn file produced by the tools and one produced by the game have the same
layout — only the content differs, and the game resolves the placeholders on first read.
This is the cleanest example in the chapter of what "compiled twice with different macro
sets" costs: **the layout is invariant, the content is not.**

### Save read — the version changelog

The whole mixin is absent below version 20.

| From version | Field |
|---|---|
| < 108 | a 32-bit word that must be zero, consumed and discarded |
| < 36 | a list of 16-bit values, consumed and discarded |
| > 62 | money |
| 76–97 | the individual as a **dense index**, converted to an identifier through the profile index |
| ≥ 98 | the individual as a string |
| > 77 | the flag word |
| 82–95 | the profile as a **dense index**, converted to an identifier |
| > 95 | the profile as a string |
| > 85 | the faction index |
| > 86 | rank and reputation |
| > 104 | the display name |
| > 124 | the two corpse bytes |

**Invariants** — the index-to-string migration at versions 96 and 98 is the most instructive
entry here. Both the profile and the individual were once stored as a position in the
authored file set, which meant **adding one profile to the middle of a file invalidated every
existing save**. The fix was to store the name. A rebuild that reads old saves must keep the
index path and the index it resolves against — see
[`xml_str_id_loader.h`](xml_str_id_loader.h.md); a rebuild that only writes new saves stores
names and is done.

**The version-108 gate asserts the discarded word is zero** rather than merely skipping it.
It is a field whose meaning is not recorded anywhere; the assertion is the only evidence
that it was always zero in practice.

**Profile resolution is triggered from inside the read** in a game build, so a record
arrives from a save already carrying a resolved individual. In the tools build it is not,
and the record stays unresolved.

### `init` — the configuration append

**Contract** — appends a `game_info` section header to the record's per-entity configuration
text. Runs at construction.

**Notes** — the append exists so that a later read of `game_info` finds a section even when
the author wrote none. The implementation picks between a stack buffer and a heap string at
a threshold of 4096 characters — an allocation choice with no bearing on behaviour. What
*is* load-bearing: a record's per-entity configuration is text that can be appended to, and
something downstream depends on that section existing.

### `specific_character` — resolving a profile into an individual

The most consequential algorithm in the file. A record names a **profile**, which may be
either a direct reference to one individual or a *template* describing a kind of person.
This turns it into exactly one individual.

```text
FUNCTION resolve_individual(record) -> id
  IF this is a multiplayer match
    RETURN whatever is stored, resolving nothing     # multiplayer has no alife registry
  IF an individual is already resolved
    RETURN it

  profile = load_profile(record.character_profile)
  IF profile names an individual directly
    adopt(that individual); RETURN it

  # a template: gather candidates, then choose
  candidates = empty; defaults = empty
  FOR EACH individual IN every authored individual
    IF individual is excluded from random choice
      CONTINUE
    IF profile names a class AND individual does not belong to it
      CONTINUE
    IF individual is its faction's default
      APPEND individual TO defaults
    IF profile fixes a rank AND |individual.rank - profile.rank| >= RANK_DELTA
      CONTINUE
    IF profile fixes a reputation AND |individual.reputation - profile.reputation| >= REPUTATION_DELTA
      CONTINUE
    IF individual is already in use by another entity
      CONTINUE                                       # each individual appears once per world
    APPEND individual TO candidates

  REQUIRE defaults is non-empty, naming the class    # a class with no default is a data error
  IF candidates is empty
    adopt(uniform random choice from defaults)       # reuse a default rather than fail
  ELSE
    adopt(uniform random choice from candidates)
  RETURN the adopted individual
```

**Invariants** — `RANK_DELTA` and `REPUTATION_DELTA` are both **10**, on scales that run to a
few hundred. No derivation is recorded; they are tuning, and they are the kind of number a
rebuild may change freely as long as it changes both consistently.

**An individual is used at most once in a world.** The in-use check reads a registry the
alife simulation keeps, and adoption adds to it. That is what stops three stalkers in one
camp from all being the same named character. It is also why the fallback to the defaults
list exists: once the pool is exhausted the world must still populate, and repeating a
default is judged better than failing.

**The defaults list is gathered before the rank and reputation filters**, so an individual
who is their faction's default counts as a default even when their rank is far from the
profile's. That ordering is deliberate and easy to get wrong when re-deriving the loop.

**The tools build never consults the in-use registry** and always chooses from the defaults,
so a spawn file's authored individuals are deterministic while a game's are not.

**The multiplayer short-circuit is marked in the source as a hack to be removed.** It is
load-bearing: multiplayer has no alife registry at all, so every step below it would fail.

### `set_specific_character` — adopting an individual

**Contract** — releases the previously adopted individual from the in-use registry, claims
the new one, then **writes the individual's authored facts into the record**. Side-effecting
and order-dependent.

```text
FUNCTION adopt(record, id)
  REQUIRE id is non-empty
  IF an individual was already adopted
    release it from the in-use registry
  record.specific_character = id
  claim id IN the in-use registry
  individual = load_individual(id)

  IF individual has a visual
    record.visual = individual.visual          # via the record's visual facet
  IF record.faction is unset
    record.faction = individual.faction
    IF the record is a creature
      record.team = individual.faction's team
  IF the record is a monster AND individual names a terrain section
    record.terrain = parse_terrain(that section)
  IF record.rank is unset       — record.rank = individual.rank
  IF record.reputation is unset — record.reputation = individual.reputation
  record.portrait = individual.portrait
  record.display_name = localize(individual.name)
  IF display_name begins with the name-generation marker
    record.display_name = generate_name(the marker's suffix)
  IF individual's money range is non-degenerate
    record.money = min + uniform random in [0, max - min)
```

**Invariants** — **"unset" is what makes a record's own value win.** Faction, rank and
reputation are filled from the individual only when the record left them at their unset
marker, so an author who overrode a stalker's rank in the spawn file keeps that override.
The visual, the portrait and the name are *not* guarded this way and always come from the
individual.

**Name generation** is a small sub-algorithm worth stating: a display name of the form
`GENERATE_NAME_<subset>` is replaced by a random first name and a random surname drawn from
a configuration section named after the subset, each section declaring how many of each it
holds. This is how a hundred anonymous stalkers get plausible Slavic names from a dozen
lines of configuration.

**Money is drawn from a half-open range** — the maximum is never itself produced, because
the random draw is over the difference. A one-off with no consequence beyond making the
authored maximum unreachable.

**In the tools build, adoption ends by clearing the individual again**, deliberately: the
editor must show an unresolved profile so the author sees the template rather than one
arbitrary resolution of it.

### Network update

Empty in both directions. Identity does not change per tick.

## `CSE_ALifeTrader`

A dynamic visual object plus the identity mixin. Both halves serialize in order.

**Contract** — not interactive: the player cannot use it directly, only trade through the
dialogue system.

**Save read** — after both halves, three legacy blocks are consumed and discarded:

- versions 36–117: one 32-bit word;
- versions 30–117: a nested inventory listing — a count, then per entry a name, a word, and a
  nested count of (name, word, word) triples;
- versions 31–117: a price list — a count, then per entry a name, a word and two floats.

**Notes** — all three are a **trader's stock and price table**, which used to live in the
record and moved into script at version 118. The read path must still walk them to stay
aligned; nothing is kept. This is the single largest discarded region in the chapter and a
good illustration of the rule that a save reader is a parser for every format version the
game ever had, not just the current one.

**A debug build refuses to validate this record's configuration when a designer flag is on
the command line**, which is how designers load levels with deliberately incomplete data.

## The zones

### `CSE_ALifeCustomZone`

A restrictor volume that does something to what is inside it.

```text
RECORD CustomZone EXTENDS SpaceRestrictor
  max_power         : real       # from the section's min_start_power, default 1
  hit_type          : enum       # from the section; "no hit type" when absent
  owner_id          : int (32-bit)   # unset is all-ones
  enabled_time      : int (32-bit)   # seconds on
  disabled_time     : int (32-bit)   # seconds off
  start_time_shift  : int (32-bit)   # phase offset into the cycle
```

**Invariants** — the three timing fields together are a **duty cycle**: a zone with a
non-zero disabled time switches itself on and off forever, and the shift staggers zones so
that a field of anomalies does not pulse in unison.

**Save read** — power always; below 113 a float and a word discarded; 67–117 a word
discarded; above 102 the owner; above 105 the two durations; above 106 the phase shift.

**Save write** — power, owner, and the three timings.

### `CSE_ALifeAnomalousZone`

```text
RECORD AnomalousZone EXTENDS CustomZone
  offline_interactive_radius : real    # default 30
  artefact_spawn_count       : int (16-bit)   # default 32
  artefact_position_offset   : int (32-bit)
```

**Invariants** — **construction sets the "destroy on spawn" flag**. An anomalous zone is
authored in the level but is not meant to persist as itself; it spawns its artefacts and
goes away. The editor property list exposes the spawn-place count with a range starting at
the default, which says the count is a *budget*, not a tuning knob.

**Save read** — a dense thicket of discarded legacy regions, worth listing because it is the
worst case in the chapter and a rebuild must reproduce the skipping exactly:

| Version range | What is consumed |
|---|---|
| > 21 | the offline radius (kept) |
| > 21 and < 113 | a float, then a counted list of (name, and a float above version 26 / a word at or below it) |
| > 25 | the artefact count and position offset (kept) |
| 28–66 | one word |
| 39–112 | one float |
| 79–112 | three floats |
| **exactly 102** | one extra word |

**Notes** — **the version-102 special case is annotated in the source with an expletive and
nothing else.** A single format version wrote one extra word; whoever found it left no
explanation, and none is recoverable. It must be reproduced.

**Three evaluation-function type numbers** are answered by reading the section rather than
from stored state, and the creature-type one **delegates to the weapon-type accessor of the
level above** — almost certainly a copy-paste defect, but one that ships and that any
behaviour comparison would reproduce.

### `CSE_ALifeTorridZone`

An anomaly on a motion path. Its payload is the base zone followed by the authored motion.
Reading it marks the record's motion as changed so the editor re-evaluates the path.

### `CSE_ALifeZoneVisual`

An anomaly with a model. Payload: the base zone, the visual, the startup animation name and
an attack animation name. The editor picks the attack animation from the visual's own
skeleton.

## `CSE_ALifeCreatureAbstract` — the level that is alive

```text
RECORD CreatureAbstract EXTENDS DynamicObjectVisual
  team, squad, group   : int (8-bit each)    # the three-level allegiance
  health               : real                # 0..1 from version 115; 0..100 before
  killer_id            : entity id (16-bit)  # all-ones when nobody
  game_death_time      : int (64-bit)        # game time of death
  dynamic_out_restrictions : list<entity id> # restrictors added at run time
  dynamic_in_restrictions  : list<entity id>
  # update-only, never saved:
  timestamp            : int (32-bit)        # server game time of this update
  flags                : int (8-bit)
  model_yaw            : real
  torso                : (pitch, yaw, roll)
  # configuration-derived, never saved:
  ef_creature_type, ef_weapon_type, ef_detector_type : int (32-bit)
  morale, accuracy, intelligence : real
```

**Invariants**

- **Alive means health above zero**, everywhere. There is no separate flag.
- **Health and killer must agree**: setting a positive health while a killer is recorded is
  a hard failure. A record that was killed stays killed.
- A creature **may not switch offline while alive is false** — a corpse stays loaded so the
  player can loot it — which is the one place the online/offline rule depends on a
  creature-specific fact.
- The two evaluation-function accessors for weapon and detector type **require** that the
  section set them; only the creature type has a mandatory configuration key. A creature
  whose section omits a weapon type fails the moment something asks.

**Save write** — team, squad, group, health, the two restriction lists, the killer, the
death time.

**Save read** — team, squad and group always; health above 18; **health divided by 100 below
version 115**; the visual below 32; the two restriction lists above 87; the killer above 94;
the death time above 115.

**Notes** — **the health rescale at version 115** is the single most important line in this
record. Health was a percentage and became a fraction. A rebuild that misses it loads every
pre-115 save with creatures at a hundred times their health, which does not crash and does
not look wrong until something takes damage.

**Orientation is reconstructed, not read.** The model yaw is taken from the torso yaw, and
the torso pitch and yaw are taken from the record's own placement angles, after the read.
Orientation therefore has exactly one authority — the placement — and the torso fields are a
derived cache. On the wire they are separate and authoritative; in a save they are not.

**Network update** — health, timestamp, flags, position, then the model yaw and the three
torso angles **as full floats**. The source carries the original quantized calls commented
out beside each one: this update was once four bytes of angles and is now sixteen. No
reason is recorded. It is the most obvious available saving in the whole protocol and the
comment is the only evidence anyone considered it.

## `CSE_ALifeMonsterAbstract` — the level that is scheduled

Adds offline scheduling, offline movement and a brain.

```text
RECORD MonsterAbstract EXTENDS CreatureAbstract, Schedulable, MovementHolder
  out_space_restrictors : text     # names, not identities: resolved at spawn
  in_space_restrictors  : text
  smart_terrain_id      : entity id (16-bit)   # all-ones when unassigned
  task_reached          : bool
  # configuration-derived, never saved:
  max_health, retreat_threshold, eye_range, hit_power : real
  hit_type              : enum
  immunity_factors      : list<real>   # one per hit type
  rank                  : int
  stay_after_death_interval : time
  group_id              : entity id (16-bit)
  terrain               : list<mask>
```

**Invariants**

- **The smart-terrain assignment and `task_reached` together are a state machine**, and the
  source says so in a comment: assigned but not reached means *walking there*; assigned and
  reached means *working there*; unassigned means idle. Two fields, three states, and
  nothing enforces the fourth combination is impossible.
- **Restrictors are stored by name and resolved later.** A record can name a restrictor that
  does not exist yet, which is what makes an authored spawn file order-independent.
- **Immunity factors are indexed by hit type**, one per type, defaulting to 1 and read from
  either the record's own section or a separate section it names. Sharing an immunity
  section across many creature sections is how the game gives a whole species one damage
  profile.
- A monster **is not updated while its health is at or below zero**, on top of the generic
  scheduling condition.

**Construction** reads a long list of tuning out of the section: going speed and its
per-level override, the terrain preference, maximum health, hit power and type, the immunity
table, the retreat threshold, the eye range, the rank, and the corpse-persistence interval.
The offline movement position is initialized to "standing still at the current graph
vertex".

**`init`** additionally lets the **per-entity configuration override the terrain
preference** — an authored entity can differ from its species. Then it creates the brain,
through an overridable step, which is where a human diverges.

**Save write** — the two restrictor name strings, the smart-terrain identity, then
`task_reached`.

**Save read** — the out-restrictors above 72, the in-restrictors above 73, the smart terrain
above 111, `task_reached` above 113.

**Notes** — **`task_reached` is written two different ways depending on the stream.** Against
a configuration stream it is a 16-bit number; against a binary stream it is the raw
in-memory boolean. That is the editor's configuration-backed serialization diverging from
the binary one (see the chapter opener), and it means a record round-tripped through
configuration is not byte-equal to one round-tripped through a packet. A rebuild should pick
one width — the value is one bit — and must then accept that the editor's files change.

**Network update** — the two graph vertices and the two distances from the offline movement
position. This is the whole of what an offline monster's motion looks like on the wire, and
it is four small fields: where it came from, where it is going, and how far along.

**Killing a monster** unregisters it from its squad before zeroing its health, so a squad
never holds a dead member.

## `CSE_ALifeCreatureActor` — the player

A creature, a trader and a ragdoll at once.

```text
RECORD Actor EXTENDS CreatureAbstract, TraderMixin, PhysicsSkeleton
  holder_id      : entity id (16-bit)   # the vehicle being driven; all-ones when none
  # update-only:
  movement_state : int (16-bit, bit field)
  acceleration   : direction+magnitude
  velocity       : direction+magnitude
  radiation      : real
  active_weapon  : int (8-bit)
  bone_count     : int (16-bit)
  alive_state    : rigid-body snapshot
  corpse_bones   : bytes                # a fixed 1024-byte buffer
```

**Save read** — a full alternate path **below version 21**, where the actor was not yet a
creature: the dynamic-object level, then team, squad and group, then health above 18, then
the visual at or above 3. At and above 21 it is the normal composition. Then the physics
skeleton above 91 and the vehicle above 88.

**Notes** — this is the only record in the chapter with a **wholesale alternative read path**
rather than field-level gates. Version 21 is where the actor was folded into the creature
hierarchy.

**Network update** — the creature and trader halves, then movement state, acceleration and
velocity as **direction-plus-magnitude** (a 16-bit direction and a float), radiation, the
active weapon slot, and then a bone count that selects between two entirely different
payloads:

```text
IF bone_count == 0
  nothing follows                         # the actor is not physically simulated
ELSE IF bone_count == 1
  one rigid-body snapshot: enabled flag, angular and linear velocity,
  force, torque, position, and the four components of the orientation
ELSE
  a corpse: one byte of per-bone record size, then 24 + size * bone_count raw bytes
```

**Invariants** — **the corpse payload is a raw byte blob with a per-bone stride** and a
24-byte header the layout of which is documented only in a source comment: six floats giving
the quantization bounds, followed by seven bytes per bone (position and rotation). Nothing
validates that the blob fits the fixed 1024-byte buffer, so a corpse with more than about
140 bones overruns it. The bone count is attacker-controlled on a multiplayer client. A
rebuild must bound it.

**The corpse read path logs a dismissive Russian message and reads the blob anyway** — the
path is half-abandoned. Treat the alive path as the real one and the corpse path as
something to design properly rather than reproduce.

## `CSE_ALifeCreatureCrow` / `CSE_ALifeCreaturePhantom`

Creatures deliberately excluded from two systems: they **do not use navigation-graph
locations** and they **never switch offline**, both set at construction. A crow's save read
is additionally gated on version 21 in its entirety — below that, nothing is read at all.

**Notes** — "does not use ai locations" is the record saying it is not on the navigation
graph. A crow flies; a phantom is an illusion. Both would otherwise be snapped to a graph
vertex they have no business occupying.

## `CSE_ALifeMonsterRat` / `CSE_ALifeMonsterZombie`

Two records that carry **their entire behavioural tuning in the record instead of in the
configuration section**: field of view, eye range, three speeds, pursuit and home radii, a
morale model of eight fields (rat only), and four or five attack parameters.

**Invariants** — every one of these is written to and read from the save unconditionally, in
a fixed order, with no version gates except one: **health was read from the payload at or
below version 5** and is inherited from the creature level thereafter. The defaults are hard
coded in the constructor rather than read from the section.

**Notes** — this is the pattern the rest of the chapter deliberately avoids, and it is worth
naming as a mistake: **because the tuning is in the record, a patch that rebalances rats
cannot affect any existing save**, and the editor must expose twenty-odd numeric rows per
rat. Every later creature puts its tuning in the section and stores only what actually
varies. The rat and the zombie are the two that predate that decision. A rebuild should
reproduce the layout — the saves demand it — and should not extend the pattern.

The rat is additionally an **inventory item**, because its corpse can be carried. Its
"useful" test — is this worth keeping around — is "not part of a group, and dead", which is
the only place a record answers that question about itself.

## `CSE_ALifeMonsterBase`

Every other creature: a monster with a ragdoll and one extra field, a **special object
identity** used by monsters that reference another entity (a controller's puppet, a burer's
target). Save read gates the ragdoll at version 68 and the special object at 109. The
supply-spawning hooks are overridden to do nothing — a monster carries no loot.

## `CSE_ALifePsyDogPhantom`

A monster that answers **no** to "are you a candidate for the simulation's attention". It
exists only while its summoner does; nothing should schedule around it.

## `CSE_ALifeHumanAbstract`

A trader **and** a monster, with the human brain in place of the monster one. Its payload is
the trader half, the monster half, and then **the brain's own state** — see
[`alife_human_brain.cpp`](alife_human_brain.cpp.md), which carries its own version gates for
all three games' saves.

**Save read** — plus one legacy gate: versions 110 and 111 wrote the smart-terrain identity
*again*, as raw bytes, after the brain. **Network update** — plus three discarded 32-bit
words below version 110.

**Notes** — the inheritance order matters and is visible in the payload: the trader half
comes **first**, before the monster half, which is the reverse of the actor's ordering where
the creature half comes first. Two records mixing the same two things in opposite orders
produce different bytes, and both orders ship.

## `CSE_ALifeHumanStalker`

A human with a ragdoll and a **starting dialogue** name.

**Invariants** — construction sets the infinite-ammunition flag. Every stalker in the game
has unlimited reserve ammunition; the flag exists so the exception is expressible.

**Notes** — the starting dialogue is written in the **network update**, not in the save. It
is the only string in the creature family that travels per tick, and it is almost certainly
in the wrong serialization: it never changes. Save read gates the ragdoll at 68 and discards
one byte for versions 91–110.

## `CSE_ALifeOnlineOfflineGroup` — the squad

A record whose state is a **set of member identities**. It has a brain and a movement
position of its own, and it moves as one thing while its members are offline.

```text
RECORD OnlineOfflineGroup EXTENDS DynamicObject, Schedulable, MovementHolder
  members : map<entity id, member record>   # the record side is not saved
```

**Save write** — the member count, then each member's identity.

**Save read** — the count, then each identity, each inserted with **no record attached**.
The records are bound later, when the world has been fully loaded and every identity can be
resolved.

**Invariants** — **a group does not use navigation-graph locations**, set at `init`: the
group is a fiction whose position is its members' collective one.

Destruction unregisters every member first, so a member never outlives its group's
knowledge of it. A member's death notifies the group, which is how a squad shrinks.

**Notes** — the group **has no current task of its own**: asking for one is a hard failure
by construction. A group's task comes from its commander. That is a deliberate hole in an
interface the level above declares, and it is the clearest statement in the chapter that a
squad is not a creature.

**The save write and read are both wrapped in an always-true build condition**, the residue
of the member list having once been optional. Nothing to preserve.
