# src/xrGame/inventory_item.cpp

> What it means to be a carryable thing: a name and a weight read from configuration, a condition that wears down, a place in someone's inventory, a physical body when nobody is holding it, and — in multiplayer — a stream of network samples to interpolate between.

**Needs** — [`inventory_item.h`](inventory_item.h.md) · [`inventory_item_impl.h`](inventory_item_impl.h.md) · [`inventory_item_inline.h`](inventory_item_inline.h.md) · [`Inventory.h`](Inventory.h.md) · [`PhysicsShellHolder.h`](PhysicsShellHolder.h.md) · [`entity_alive.h`](entity_alive.h.md) · [`Actor.h`](Actor.h.md) · [`Level.h`](Level.h.md) · [`game_cl_base.h`](game_cl_base.h.md) · [`hit_immunity.h`](hit_immunity.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [`xrAICore/Navigation/ai_object_location.h`](../xrAICore/Navigation/ai_object_location.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics) · [Seam: Configuration format](../xrCore/README.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: writes and reads a frozen wire format field by field, and hands a physics body raw state

## Purpose

This is the base every carryable thing in the game shares, and it is doing four jobs at once
because an item genuinely is four things depending on where it is:

1. **A configuration-driven record** — name, weight, cost, icon, slot. Read once at load.
2. **A thing in a grid** — with a place, a condition, and permissions about being taken,
   traded and dropped.
3. **A physics object** — when nobody is holding it, it is a rigid body lying in the world.
4. **A network-replicated object** — in multiplayer, a remote client receives samples of that
   body's state and must interpolate between them.

The single most confusing thing about the file is that job four and job three interact badly
and the source records three abandoned attempts at reconciling them. What ships is the
simplest of the four approaches, and the twin says so rather than describing machinery that
is commented out.

## State

See [`inventory_item.h`](inventory_item.h.md) for the flag set and the network sample record.
The condition starts at one — undamaged — and is clamped to the unit interval forever after.

## Construction and defaults

**Contract** — a fresh item is **in the backpack, not on the belt, defaults to the backpack,
can be taken, can be traded, does not wear out, and is not a helper item**. Its place is
undefined with no slot.

**Invariants** — "defaults to the backpack" is the important one: an item with no configured
slot goes into the backpack rather than being unplaceable, so an authored item that forgot to
declare a slot is still pickable. Every other default is permissive for the same reason —
data errors produce a usable item rather than an invisible one.

## `Load`

**Contract** — reads the item's tuning from its configuration section. Required keys fail
loudly; optional ones have the permissive defaults above.

```text
FUNCTION load(section)
  load the damage-immunity table from the section named by "immunities_sect"
  mark this object visible to creature perception      # items must be seeable by AI
  name       = translate(section."inv_name")
  short_name = translate(section."inv_name_short")
  weight     = section."inv_weight"      REQUIRE weight >= 0
  cost       = section."cost"
  base_slot  = section."slot" (default -1) + 1         # see below
  description = translate(section."description") or empty
  belt, can_trade, can_take, quest_item, use_condition, highlight_equipped : optional bools
  IF the item has a slot OR goes on the belt THEN
    ruck_default        = section."default_to_ruck"        (default true)
    allow_sprint        = section."sprint_allowed"         (default true)
    control_inertion    = section."control_inertion_factor"(default 1)
                          # forced to 1 when mouse sensitivity normalization is on
  icon_name = section."icon_name" (optional)
```

**Invariants**, and the first is the one that will bite a rebuild:

- **The slot number is shifted by one on read.** The shipped game data numbers slots from
  minus one (minus one meaning "no slot"); the engine numbers them from zero. Every slot in
  every configuration file is therefore one less than the engine's. A rebuild that reads the
  data unshifted will put every item in the wrong slot and will place slotless items in slot
  zero.
- **Three keys are read only for items that have a slot or a belt position.** A backpack-only
  item has no sprint rule and no control inertion, because those describe the item while it is
  *held*, and it never is. A rebuild that reads them unconditionally gets the same behaviour
  and wastes nothing, but must not then act on them for backpack items.
- **The immunity section is a separate named section**, not this item's own. Many items share
  one immunity profile, and the indirection is how.
- **Names and descriptions are translated at load**, so the item holds display text, not
  identifiers. That is why a language change needs an explicit reload.

## `ReloadNames`

**Contract** — re-translates the name, short name and description from the item's own section.
Called when the language changes.

**Invariants** — reads the section from the live object rather than from a stored section name.
The two agree; a rebuild should use one of them.

## Condition

### `ChangeCondition`

**Contract** — adds a signed delta and clamps to the unit interval. The *only* way condition
changes.

### `Hit`

**Contract** — damage to an item degrades it, **but only if the item wears out at all**.

```text
FUNCTION on_hit(hit)
  IF NOT use_condition THEN RETURN
  change_condition( - hit.damage * immunity_for(hit.type) )
```

**Invariants** — the item's own damage-immunity table scales the wear, so the same blast
degrades an armoured outfit less than a torch. Items that do not declare wear are immune to
damage entirely regardless of their immunity table.

## Ownership transitions

**Contract** — four hooks, fired as an item changes hands. The two that do real work:

- **Becoming independent (before)** — recompute the item's world transform from its former
  owner's skeleton, then mark the place undefined. The order matters: the transform must be
  taken while the owner link still exists.
- **Becoming independent (after)** — record the moment of independence on the server clock, and
  mark the place undefined again.
- **Becoming a child (before)** — remove the item from the level's physics correction set. A
  carried item is not independently simulated.
- **Becoming a child (after)** — delegate.

**Invariants** — the independence timestamp is what the multiplayer cleanup rule measures
against; see `NeedToDestroyObject`.

## `UpdateXForm`

**Contract** — computes where a carried item actually is in the world, from its holder's hand
bones. Used when the item is about to become independent, or to be rendered in the world.

```text
FUNCTION update_transform()
  IF no parent THEN RETURN
  IF the parent is not a living entity THEN RETURN
  IF the parent is a base monster THEN RETURN          # monsters have no weapon bones
  IF the parent is an inventory owner that has this item ATTACHED THEN RETURN
                                                       # an attachment has its own transform

  (left_bone, right_bone, secondary_bone) = parent.weapon_bones()
  IF there is no right bone THEN RETURN

  parent.skeleton.calculate_bones()                    # forced, every call
  left  = left bone's transform
  right = right bone's transform

  forward = normalize(left.origin - right.origin)      # the item points from the RIGHT
                                                       # hand toward the LEFT
  IF forward is degenerate THEN
    result = parent's transform with its origin replaced by the right hand's
  ELSE
    right_axis = cross(right_bone.up, forward)
    up_axis    = normalize(cross(forward, right_axis))
    result     = basis(right_axis, up_axis, forward) at the right hand's origin
    result     = result composed with the parent's transform

  item.position = result.origin
```

**Invariants** — the item's orientation is derived from the **line between the two hands**, not
from either hand's own orientation. A two-handed weapon is therefore aimed correctly even
when the animator rotated one hand for grip; a one-handed item falls back to the parent's
orientation because the two hands coincide.

Only the **position** is written back, not the orientation. The orientation is recomputed but
discarded. That is the shipped behaviour and a rebuild should reproduce it before
"fixing" it: the orientation is supplied elsewhere, by the attachment machinery, and writing
it here fights that.

**Notes** — the forced skeleton evaluation is flagged in the source as a serious performance
problem, and it is: this runs per item per relevant frame and forces a full pose evaluation of
the holder. A rebuild should cache the pose per holder per frame.

## Detaching by section

**Contract** — detaching a sub-item (a scope, a silencer) does not move an object — it
**spawns a new one** into the owner's inventory.

```text
FUNCTION detach(section_name, spawn_it) -> bool
  IF running on a client THEN RETURN true        # the server does this; the client waits
  IF spawn_it THEN
    record = create a server record for `section_name`
    record.navigation_vertex = this item's vertex, or none on a dedicated server
    record.section  = section_name
    record.parent   = this item's parent
    record.position = this item's position
    record.flags    = locally spawned
    broadcast the spawn ; destroy the temporary record
  RETURN true
```

**Invariants** — this is the engine's general pattern for "a thing becomes a different thing":
never mutate, always spawn a new server record and let the normal spawn path create the client
object. It is why every detachable attachment has its own configuration section.

A dedicated server writes no navigation vertex, because it has no level graph loaded.

In multiplayer the parent may legitimately be absent — an item being detached inside the buy
menu has no carrier — and the record then claims parent zero rather than none. The source
marks this as a workaround, and it is: parent zero is a valid entity identifier. A rebuild
should use the none sentinel.

## Spawn and destroy

### `net_Spawn`

**Contract** — initializes from the server record. Must be called on an item not yet in an
inventory.

```text
FUNCTION net_spawn(server_record)
  REQUIRE this item is not in an inventory
  clear the interpolation flags
  useful_for_npc = server_record.useful_for_ai  (default true if not an alife object)
  IF the record is not an inventory item THEN RETURN success   # nothing else to take

  condition = server_record.condition
  IF this is not a single-player game THEN enable per-frame processing
  independence_time = 0
  mark just-spawned, not yet activated for interpolation
```

**Invariants** — the condition comes from the server record, which is the authoritative copy;
an item's wear survives save/load and level change because of this one line.

Per-frame processing is enabled **only in multiplayer**, because only multiplayer needs the
interpolation that processing drives. In single player an item lying on the ground costs
nothing per frame.

### `net_Destroy`

**Contract** — asserts the item is no longer listed in any inventory. Deliberately does *not*
clear the inventory reference; the source's commented-out line shows that was considered and
rejected, because destruction order lets other code still need the link.

## Persistence

### `save`

**Contract** — writes the place, the condition, and then **either nothing or a full physics
state**, depending on whether the item is carried.

```text
FUNCTION save(packet)
  packet.write(place)          # 16 bits: type, base slot, current slot
  packet.write(condition)
  IF the item has a parent THEN
    packet.write_byte(0)       # carried: the parent's save records where it is
    RETURN
  packet.write_byte(number of physics sync elements)
  write the full physics state
```

**Invariants** — a carried item saves no position. Its position is its carrier's, and writing
one would restore a dropped item to a stale place after a carrier moved.

The count byte doubles as a presence flag: zero means "carried, no physics follows". The load
path relies on that.

**Notes** — the upgrade list is **not saved**; the calls are present and commented out. Upgrades
are recovered from the server record instead, which is why the upgrade installation happens in
the spawn path (see [`inventory_item_upgrade.cpp`](inventory_item_upgrade.cpp.md)) rather than
here.

### `load`

**Contract** — reads place and condition, then, if a physics state follows, **creates a physics
body if the item does not have one** and loads the state into it, leaving the body disabled.

**Invariants** — the body is disabled both immediately after creation and again after loading
the state. Loading state can wake a body; a restored item must be asleep until something
disturbs it, or every loose object in the level starts settling the moment a save is loaded.

## Network replication

### `net_Export`

**Contract** — writes the item's physics state, or a single zero byte when there is nothing to
send.

```text
FUNCTION net_export(packet)
  IF the item has a parent OR this is single player THEN
    packet.write_byte(0) ; RETURN          # carried items are replicated by their carrier

  state = the physics body's state, or just the position when there is no body
  mask.element_count = number of sync elements      REQUIRE it fits in 5 bits
  IF state.enabled           THEN mask |= enabled
  IF angular velocity is zero THEN mask |= angular_null
  IF linear velocity is zero  THEN mask |= linear_null

  packet.write_byte(mask and count packed into one byte)
  IF that byte is zero THEN RETURN
  write force, torque, position, orientation
  write angular velocity   UNLESS angular_null
  write linear velocity    UNLESS linear_null
  packet.write_byte(is the body awake)
```

**Invariants** — the first byte packs a five-bit element count with three flag bits. That
packing is the frozen wire format; the assertion that the count fits in five bits is the only
guard, and it is a hard failure because exceeding it would silently corrupt the flags.

The two "velocity is zero" flags are a real saving: most replicated items are at rest, and the
flags elide six floats each.

A zero-magnitude orientation is replaced with a fixed unit quaternion rather than transmitted,
because a zero quaternion applied on the far side produces a degenerate transform. The
normalization that would have accompanied it is disabled, so orientations are sent unnormalized
and the receiver must cope.

### `net_Import`

**Contract** — reads a sample, timestamps it with the local clock, and appends it to the sample
queue — **unless this client owns the object**, in which case the sample is read and discarded.

```text
FUNCTION net_import(packet)
  count_and_mask = packet.read_byte()
  IF zero THEN RETURN
  sample.timestamp = local clock now
  read force, torque, position, orientation, and the two velocities per the mask
  sample.previous_position = sample.position           # a fresh sample has no history
  sample.previous_orientation = sample.orientation
  packet.read_byte()                                   # awake flag, read and ignored

  IF this client owns the object THEN RETURN           # we are authoritative; do not
                                                       # interpolate toward our own echo
  add the object to the level's physics correction set
  append the sample
  WHILE more than two samples are queued: drop the oldest
  IF not yet activated THEN enable per-frame processing and mark activated
```

**Invariants**:

- **The sample is stamped with the receiver's clock, not the sender's.** The interpolation
  therefore measures against local time and is immune to clock skew, at the cost of being
  sensitive to jitter in arrival times. That is the trade the shipped engine makes.
- **At most two samples are kept.** Interpolation needs exactly a span; a longer history would
  buy smoothing at the cost of latency, and the engine chooses latency.
- **Per-frame processing is enabled lazily**, on the first sample that will actually be
  interpolated, and disabled again when the queue drains. A replicated item that has stopped
  moving costs nothing.
- The awake flag is transmitted and discarded on receipt. A rebuild may stop sending it, but
  only by changing the format on both sides.

### `Interpolate` and `interpolate_states`

**Contract** — the shipped interpolation: **linear in position, spherical in orientation,
between the two queued samples, extrapolating past the newer one**.

```text
FUNCTION interpolate()
  IF the item has a parent, is invisible, has no body, or we are the server THEN RETURN
  IF no samples THEN RETURN
  state = the OLDER sample's state
  IF two samples are queued THEN
    factor = interpolate(older, newer, into state)
    IF factor >= 1 THEN
      drop the older sample
      IF activated THEN disable per-frame processing and mark deactivated
  body.state = state

FUNCTION interpolate(first, last, out) -> real
  IF now equals last.timestamp THEN RETURN 0
  factor = (now - last.timestamp) / (last.timestamp - first.timestamp)
  out.position    = lerp(first.position, last.position, clamp(factor, 0, 1))
  out.orientation = slerp(first.orientation, last.orientation, clamp(factor, 0, 1))
  out.previous_* = out.*
  RETURN factor          # UNCLAMPED: the caller uses it to detect arrival
```

**Invariants** — the factor is computed as elapsed-since-the-**newer**-sample divided by the
span between the two, which means it is **zero at the newer sample's arrival and one after a
further full span**. It is an *extrapolation* parameter, not an interpolation parameter: the
item is rendered ahead of the newest data by up to one inter-sample interval. That is
deliberate — it hides one packet interval of latency — and it is why the value returned is
unclamped while the value used is clamped: the caller needs to know it passed one to retire
the sample, and the blend must not be allowed past the endpoint.

The previous position and orientation are overwritten with the current ones on every step, so
the physics layer computes no velocity from the interpolation. A replicated item is positioned,
not simulated.

**Notes** — three progressively more elaborate schemes precede this one in the source, all
disabled: a cubic Bézier through predicted control points with velocity-derived tangents and a
duration computed from path length over speed; a full correction-prediction cycle with state
save, re-simulation and restore across four hooks; and a quantized wire format packing
orientation and velocities into bytes. The recipe records their existence because a rebuilder
will otherwise reinvent them — but what ships is the ten-line linear blend above, and it is
what the game's behaviour was tuned against.

### The correction-prediction hooks

**Contract** — four hooks the physics step calls around its correction and prediction passes.
Three are empty in the shipped build. The fourth does one thing, and it has nothing to do with
prediction:

```text
FUNCTION after_correction_prediction()
  IF NOT just_spawned THEN RETURN
  REQUIRE the item has a visual and a physics body
  IF the body is not fully active THEN
    force a full skeleton evaluation      # the body's transform is read from the bones
  item.transform = body's dynamic global transform
  force a full skeleton evaluation again  # now with the corrected transform
  update the item's spatial registration
  just_spawned = false

  REQUIRE we are not the server
  fix the body's first element in place
  make the body ignore static geometry
```

**Invariants** — this is the *first frame* fixup for a replicated item, and every step is
load-bearing:

- The skeleton must be evaluated before the body's transform is read, because for a
  not-fully-active body the transform is derived from the bones.
- It must be evaluated again after, because the item's transform just changed and the render
  and collision geometry are built from the bones.
- The spatial registration must be refreshed or the item is queryable at its spawn position
  rather than its real one.
- The body is then **pinned and made to ignore static geometry**. A replicated item is
  positioned by interpolation, not simulated; leaving it collidable would have the local solver
  fight the incoming stream and jitter. This runs only on a client — the assertion states that.

## Multiplayer item lifetime

### `NeedToDestroyObject`

**Contract** — whether a dropped item should be removed from the world.

```text
FUNCTION should_destroy() -> bool
  IF single player THEN RETURN false                 # nothing is ever cleaned up
  IF the mode is capture-the-artefact THEN RETURN false   # dropped items are the objective
  IF this object is owned by another client THEN RETURN false   # its owner decides
  RETURN time_since_independence > 30 seconds
```

**Invariants** — thirty seconds is the shipped lifetime of a dropped item in multiplayer. Long
enough to run back for a dropped weapon, short enough that a long round does not accumulate
litter. The two exemptions are both semantic: single player has an alife simulation that owns
item lifetime, and the artefact mode's whole objective is a dropped item.

### `TimePassedAfterIndependant`

**Contract** — how long since the item was dropped, on the server clock; zero while carried or
if it was never dropped.

## Permissions

### `CanTrade`

**Contract** — three conditions, all required: the owner permits it for this item in this
place, the current trade flag is set, and it is not a quest item.

**Invariants** — a quest item is never tradeable regardless of any flag. That is the hard rule;
everything else is advisory.

**Notes** — the owner is consulted only when the item is in an inventory, and the source carries
a standing question about why an ownerless item is ever asked. A rebuild should make the
question moot by requiring an owner.

### `SetDropManual`

**Contract** — marks an item as queued for dropping. **In multiplayer, additionally revokes or
restores trade permission**, and warns.

**Invariants** — restoring permission restores it to the item's configured value, not to true.
An item whose section forbids trading stays untradeable through a drop-and-cancel cycle. That
is exactly what the two-piece trade state exists for.

The warning fires on every multiplayer use, which says the mechanism is single-player
machinery leaking into a mode it was not designed for.

### `IsInvalid`

**Contract** — an item is invalid when it is being destroyed or is queued for manual drop.
Inventory code uses this as "stop treating me as present".

## Event handling

**Contract** — three network events:

- **attach addon** — resolves an item identifier and attaches it.
- **detach addon** — reads a section name and detaches by section (spawning the detached part).
- **change position** — writes a position directly into the physics state, setting both the
  current and the previous position to it so no velocity is inferred.

**Invariants** — the position event is a teleport, and setting the previous position equal to
the new one is what makes it one. A rebuild that leaves the previous position stale will have
the physics layer compute an enormous velocity from the jump.

## Small contracts

- **`Useful`** — an item is useful exactly when it can be taken. Creature inventory code asks
  this; the kinds that mean something more specific override it.
- **`ActivateItem` / `DeactivateItem`** — refuse and do nothing. Items that can be held
  override both.
- **`_construct`** — records this item's own physics-holder identity, asserted present. The
  item and the physics holder are the same object viewed two ways.
- **`reload`** — reads the two holder modifiers, both defaulting to one (no change).
- **`reinit`** — clears the inventory link and the place. Does not touch the condition or the
  upgrades, which survive a reinitialization because they belong to the item rather than to
  its placement.
- **`can_kill`** and its four relatives — all answer "no". Weapons override. They exist so a
  creature's planner can ask any item.
- **`modify_holder_params`** — multiplies a range and a field of view by this item's modifiers.
  A scope narrows the field of view and extends the range through this one call.
- **`object_id` / `parent_id`** — the item's own entity identifier, and its carrier's, or the
  all-ones sentinel when it has none.
- **The three grid rectangles** — the inventory icon's cell rectangle (required keys), the
  upgrade icon's (all optional, defaulting to zero) and the kill-message icon's (likewise).
  All are read from configuration on every call rather than cached, which is acceptable only
  because they are called from user-interface code.
- **`has_network_synchronization`** — false. Kinds that replicate more than physics state
  override it.
- **`activate_physic_shell`** — when the item's parent is a living entity, update the transform
  from the parent's hands first and then activate through the physics holder; otherwise
  delegate to the kind-specific hook, whose base implementation is a hard failure. An item
  becoming physical must know where it is before it starts falling.
