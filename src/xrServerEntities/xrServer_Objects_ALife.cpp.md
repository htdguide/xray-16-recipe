# src/xrServerEntities/xrServer_Objects_ALife.cpp

> The record layouts for everything the world is built out of that is not a creature or an inventory item: graph points, restrictors, level changers, props, lamps, vehicles, containers.

**Needs** — [`xrServer_Objects_ALife.h`](xrServer_Objects_ALife.h.md) · [`xrServer_Objects_ALife_Monsters.h`](xrServer_Objects_ALife_Monsters.h.md) · [`restriction_space.h`](restriction_space.h.md) · [`character_info.h`](character_info.h.md) · [`Common/object_broker.h`](../Common/object_broker.h.md) · [`game_base_space.h`](game_base_space.h.md) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: twenty on-disk record layouts with exact field widths and order.

## Purpose

The bulk of the spawn format. Each section below is one record: the fields it adds to its
parent's payload, in stream order, and the version gates that decide whether a field is
present. **A record's payload is always its parent's payload followed by its own** — there is
no framing between them, which is why the parent chain must be reproduced exactly even when
a level adds nothing.

Three vocabulary reminders, because they recur: *state* is the payload in the spawn file and
the save; *update* is the periodic network delta; the two are different byte sequences for
the same record and a field appears in one, the other, both, or neither.

## `CSE_ALifeObject`

The alife base: everything the off-screen simulation needs to place and switch a record.

```text
RECORD AlifeObject                     # state payload, in order
  game_vertex_id : int (16-bit)          # position on the cross-level graph
  distance       : real                  # travel distance along the current edge
  direct_control : int (32-bit)          # written as 32 bits, read as a boolean
  level_vertex_id: int (32-bit)          # position on this level's navigation mesh
  flags          : int (32-bit)          # see below
  custom_data    : text                  # the per-entity configuration overlay, verbatim
  story_id       : int (32-bit)          # script-visible persistent name; -1 = none
  spawn_story_id : int (32-bit)          # script-visible name of the spawn record; -1 = none
```

```text
CONSTANT fl_use_switches        = 1 << 0    # obey the two switch flags below at all
CONSTANT fl_switch_online       = 1 << 1
CONSTANT fl_switch_offline      = 1 << 2
CONSTANT fl_interactive         = 1 << 3
CONSTANT fl_visible_for_ai      = 1 << 4
CONSTANT fl_useful_for_ai       = 1 << 5
CONSTANT fl_offline_no_move     = 1 << 6    # note the inversion: set means "cannot move"
CONSTANT fl_used_ai_locations   = 1 << 7    # occupies a navigation vertex
CONSTANT fl_group_behaviour     = 1 << 8
CONSTANT fl_can_save            = 1 << 9
CONSTANT fl_visible_for_map     = 1 << 10
CONSTANT fl_use_smart_terrains  = 1 << 11
CONSTANT fl_check_for_separator = 1 << 12
```

**Invariants**

- **All flags start set** and each constructor clears the ones its class must not have. That
  is the opposite of the usual default and it matters: a new record type that forgets to
  clear inherits every capability.
- **The offline-movement flag is stored inverted** relative to its accessor. Scripts call
  "can this move offline"; the bit means "may not". Preserve the *bit*, not the accessor.
- `interactive` is the conjunction of three flags, not one: interactive **and** visible to AI
  **and** useful to AI. A record missing any of the three is not offered to creatures.
- `can_switch_online` is *also* gated on the record matching the current configuration and
  renderer; `can_switch_offline` is satisfied by a record that does *not* match, which is how
  a renderer-specific lamp is quietly kept offline.
- **The per-record random generator is seeded from the processor's cycle counter** at
  construction. Its consumers are all cosmetic (network-relevance thinning), but it means
  record construction is not deterministic. A rebuild wanting reproducibility should seed
  from the record's identifier instead; nothing in the format depends on the seed.

**Read gates** — before version 4 there is an extra 16-bit field; before 24 the probability
is a byte instead of a real; the spawn identifier lives *here* for versions 23 to 79 and in
the base record from 80; the custom-data string appears at 58; the story identifier at 62;
the spawn-story identifier at 112.

**Notes** — the evaluation-function type queries (equipment, main weapon, weapon, detector)
are declared here and fail hard by default with the class tag in the message. They are pure
virtuals in spirit: the base is asserting that a record reaching a creature's ranking code
must have been overridden. Only the weapon query returns a benign zero, because too many
records are asked.

## `CSE_ALifeGraphPoint`

```text
RECORD GraphPoint
  connection_point : text            # the vertex on the neighbouring level this joins
  connection_level : text            # that level's name; before v33 a 32-bit identifier
  locations        : int (8-bit) x 4 # four independent place-type axes
```

An authored vertex of the game graph. It never comes online and reports that it does not
match any configuration, so the simulation never tries. The four location bytes are the
coordinate the alife simulation uses to decide which creature belongs where; their vocabulary
is data (see the editor tables in the header twin), and the editor colours the point by the
four-byte tuple looked up in a palette.

## `CSE_ALifeGroupAbstract`

```text
RECORD GroupPopulation
  create_spawn_positions : int (32-bit)   # boolean widened
  count                  : int (16-bit)   # how many individuals this population represents
  members                : list<int (16-bit)>   # entity identifiers; from version 20
```

Its update payload is only the boolean. A population that has come online is a set of
individual records; offline it is a count and a next-birth time (the time is held but not
serialized — **unrecovered**: the field exists, is initialized to zero, and no read or write
touches it).

## `CSE_ALifeDynamicObject`

Adds a game timestamp and a switch counter to the record, **neither of which is serialized**.
Its payload is exactly its parent's. What it adds is behaviour: the right to be switched
online and offline, to be registered with the simulation, and to attach and detach inventory
children. A rebuild still needs the level as a distinct concept, because "dynamic" is the
predicate the simulation filters on.

## `CSE_ALifeDynamicObjectVisual`

Payload: parent, then the visual mixin — **but only from version 32**, which is the version at
which model serialization was centralized here. Before that, several classes wrote the model
themselves at their own offsets, and their readers still do.

## `CSE_ALifePHSkeletonObject`

Payload: parent, then the physics-skeleton mixin (from version 64). Clears the
use-switches and switch-offline flags at construction: a prop stays where the level put it.
It saves only when the skeleton mixin says it has something worth saving, and it never
occupies a navigation vertex.

## `CSE_ALifeSpaceRestrictor`

```text
RECORD SpaceRestrictor
  shapes          : Shape list        # the volume, from the shape mixin
  restrictor_type : int (8-bit)       # from version 75
```

The restrictor kind says whether the volume is not a restrictor at all, or is the default
*in* or *out* restrictor for creatures that name no other. A value outside the known range is
reported loudly and then kept, because the alternative (a hard failure) would make one bad
record unload a level; the report names the record, its game vertex and its level vertex so
the author can find it.

A restrictor may never go offline, never occupies a navigation vertex, and sets the
check-for-separator flag — it participates in deciding whether a level is split into
disconnected regions.

## `CSE_ALifeLevelChanger`

```text
RECORD LevelChanger
  next_game_vertex : int (16-bit)
  next_level_vertex: int (32-bit)
  next_position    : real x 3
  next_angles      : real x 3        # one real before version 54, three from 54
  level_to_change  : text
  point_to_change  : text
  silent_mode      : int (8-bit)     # from version 117
```

A restrictor that, when the actor enters it, moves him to a named point on a named level.
Silent mode suppresses the confirmation prompt. Before version 34 the destination was two
opaque 32-bit values, which the reader consumes and discards.

## `CSE_ALifeObjectPhysic`

A freely simulated prop, and the most involved *update* record in the file.

```text
RECORD PhysicObject                 # state payload
  ...parent (dynamic visual, and the physics-skeleton mixin from version 64)
  type        : int (32-bit)        # PhysicsObjectType; shipped data is always skeleton
  mass        : real                # kilograms; default 10
  fixed_bones : text                # space-separated bone names that do not move
```

```text
RECORD PhysicUpdate                 # the network delta
  header : int (8-bit)              # packed: low 5 bits an item count, high 3 bits a mask
  force, torque, position : real x 3
  orientation             : real x 4       # quaternion, x y z w
  angular_velocity        : real x 3       # omitted when the mask says it is zero
  linear_velocity         : real x 3       # omitted when the mask says it is zero
  awake                   : int (8-bit)    # optional trailer
```

```text
CONSTANT mask_enabled      = 1 << 0    # the body is awake
CONSTANT mask_angular_null = 1 << 1    # angular velocity omitted, treat as zero
CONSTANT mask_linear_null  = 1 << 2    # linear velocity omitted, treat as zero
```

**Invariants**

- **The count and the mask share one byte**: five bits of count (so at most 31), three of
  mask. Both sides assert the count fits. A zero byte means "no physics state follows" and
  ends the record — this is the only length signal.
- **The two velocity vectors are omitted when zero**, not written as zeros. A resting prop's
  update is 44 bytes instead of 68, and a level full of resting props is most of the update
  traffic.
- **The update must tolerate ending early.** A record may arrive as spawn-plus-update in one
  packet, in which case the trailer is absent; the reader checks for end-of-stream twice, and
  a rebuild must reproduce both checks rather than assume a fixed length.
- **The trailer byte is written as a constant 1 and the comment says it means nothing.** The
  reader nonetheless treats zero as "this body has gone to sleep" and starts the freeze
  timer. Since the writer never sends zero, the freeze path is reachable only from a peer
  that does. **Unrecovered**: whether that peer ever existed.

**`Net_Relevant`** — decides whether to send an update at all. An awake prop is always sent.
A prop that has just fallen asleep is sent once. Thereafter it is sent on roughly one frame
in 40, chosen at random, and only for five seconds after it fell asleep; after that, never.
The constants (40 draws, 5000 milliseconds) are a bandwidth tuning with no derivation —
they trade "a sleeping prop eventually agrees across clients" against traffic.

## `CSE_ALifeObjectHangingLamp`

The record with the longest history and the clearest example of a format that grew by
accretion.

```text
RECORD HangingLamp                  # current layout, version 49 and later
  colour             : int (32-bit)     # packed
  brightness         : real
  colour_animator    : text             # a named light animation
  range              : real
  flags              : int (16-bit)     # see below
  startup_animation  : text
  fixed_bones        : text
  health             : real
  virtual_size       : real             # the emitter's radius for soft shadows
  ambient_radius     : real
  ambient_power      : real
  ambient_texture    : text
  light_texture      : text
  main_bone          : text
  cone_angle         : real             # radians; spot lights only
  glow_texture       : text
  glow_radius        : real
  ambient_bone       : text             # from version 97; before that, the main bone
  volumetric_quality : real             # from version 119
  volumetric_intensity: real
  volumetric_distance: real
```

```text
CONSTANT lamp_physic        = 1 << 0    # the lamp swings and can be broken
CONSTANT lamp_cast_shadow   = 1 << 1
CONSTANT lamp_allow_r1      = 1 << 2    # valid on the forward renderer
CONSTANT lamp_allow_r2      = 1 << 3    # valid on the deferred renderers
CONSTANT lamp_type_spot     = 1 << 4    # clear means point
CONSTANT lamp_point_ambient = 1 << 5    # a second, ambient emitter
CONSTANT lamp_volumetric    = 1 << 6
```

**Invariants**

- **A lamp declares which renderer generations it is valid on**, and this is the one place in
  the chapter where a record's existence depends on a graphics decision: a level is authored
  with both a forward-renderer lamp and a deferred-renderer lamp at the same spot, and
  `match_configuration` keeps the wrong one offline. A lamp with neither bit set is rejected
  by `validate` with a message. A rebuild with one renderer must still read the bits and
  still pick one lamp, or every lit room gets lit twice.
- **Versions below 49 are a different field list entirely** — a colour-then-animator-then-two-
  discarded-strings-then-range-then-an-8-bit-angle sequence with six optional tails. It is
  reproduced field for field in the reader and is unreachable from shipped data; a rebuild
  targeting retail data may refuse below 49 and lose nothing.
- **The cone angle is stored in radians** with a default of 120 degrees converted at
  construction, and was an 8-bit quantized angle before 49.

## `CSE_ALifeCar`

```text
RECORD Car                          # state payload
  ...parent, then the physics-skeleton mixin from version 66
  health : real                     # from version 93
```

```text
RECORD CarSavedPose                 # inside the skeleton mixin's saved-data blob
  bones        : bytes              # the mixin's own pose
  position     : real x 3           # the car overwrites the record's placement from here
  angle        : real x 3
  doors        : list<(open_state : int (8-bit), health : real)>   # 16-bit count
  wheels       : list<(health : real)>                             # 16-bit count
  health       : real
```

**Invariants**

- **Health is rescaled on read**: a value above 1 is divided by 100. Car health was a
  percentage before it was a fraction, and there is no version gate — the magnitude *is* the
  discriminator. A rebuild must copy the heuristic, and must never write a value above 1.
- **The car's saved pose carries its own copy of position, angle and health**, duplicating
  fields in the record proper. On restore the pose wins. The duplication is real and a
  rebuild must write both.

## `CSE_ALifeHelicopter`

Payload: parent, then the motion mixin, then the physics-skeleton mixin (from version 69),
then the startup animation, then an engine sound name. Note that the motion comes *before*
the skeleton and the animation *after* it — an ordering no rule predicts.

## `CSE_ALifeObjectClimable`

```text
RECORD Climable
  shapes   : Shape list
  material : text          # from version 127; defaults to a fake-ladder surface material
```

A ladder: a volume plus the surface material that decides footstep sounds and hand placement.
Its parentage moved twice (versions 99 and 100) and the reader still branches on exactly
version 99 versus above — a record at 99 reads the alife-object payload, above 99 the
dynamic-object payload. It cannot go offline and occupies no navigation vertex.

## `CSE_ALifeObjectBreakable`

Payload: parent, then one real, health. Cannot go offline.

## `CSE_ALifeStationaryMgun`

The only record here whose *update* carries state its *state* does not:

```text
RECORD MachineGunUpdate
  working       : int (8-bit)
  desired_aim   : real x 3
```

An emplaced gun's aim direction is live state that need not survive a save.

## `CSE_ALifeTeamBaseZone`

Payload: restrictor, then one team byte.

## `CSE_ALifeInventoryBox`

```text
RECORD InventoryBox                 # from version 124
  can_take : int (8-bit)            # may the actor take from it
  closed   : int (8-bit)            # is it locked
  tip_text : text                   # the string-table key for the use prompt
```

The prompt defaults to a string-table key rather than a literal, as all user-facing text
must.

## `CSE_ALifeSmartZone`

A restrictor that is also schedulable — the base of smart terrain. Its payload is exactly the
restrictor's; everything it adds is behaviour, and all of that behaviour is overridden in
script or in the game module. The base answers "no job, zero probability, nothing happens on
touch", which makes a plain smart zone an inert volume the simulation still visits. It always
occupies navigation vertices.

## `CSE_ALifeSchedulable`

The mixin that makes the simulation call a record. It holds the record's currently best
weapon and best detector (neither serialized, both recomputed) and a schedule counter.

**Contract** — `need_update` is the gate: a record is updated when it is directly controlled,
occupies navigation vertices, and is **offline**. That last clause is the whole design: the
coarse simulation runs *only* on records nobody is watching, because an online record is
being simulated in detail by the game instead.

The four evaluation-function type queries (creature, anomaly, weapon, detector) all fail hard
with the class tag in the message — a schedulable record reaching the ranking code without
having overridden them is a programming error, not a data error.

## `CSE_ALifeMountedWeapon` and `CSE_ALifeObjectProjector`

Pure pass-throughs: their payload is their parent's, they add no fields. They exist to be
distinct class identifiers so the factory can build the right client object. A rebuild keeps
them as names, not as layouts.
