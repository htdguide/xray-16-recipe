# src/xrGame/ai/trader/ai_trader.cpp

> The trader's whole engine-side behaviour: turn your head toward the player, accept what is put in your hands, and let Lua do the rest.

**Needs** — [`ai_trader.h`](ai_trader.h.md) · [`trader_animation.h`](trader_animation.h.md) · [`trade.h`](../../trade.h.md) · [`relation_registry.h`](../../relation_registry.h.md) · [`Inventory.h`](../../Inventory.h.md) · [Seam: Script virtual machine](../../../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — [`ai_trader.h`](ai_trader.h.md)
**Tier floor** — T3: event handling, a head-turn rule, and delegation

## Purpose

The trader exists to show how little engine code a *conversational* non-player character
needs, once the inventory, dialogue and script-action machinery exist. Its think function is
empty. Its perception is touch only. It never paths, never aims, never chooses.

Three things here are genuinely the trader's own and a rebuild must reproduce them:

1. **The head turn.** A per-frame bone adjustment that rotates the trader's head toward the
   player inside twenty metres, clamped to one radian. It is what makes a stationary vendor
   feel present, and it is a *bone callback*, not an animation — it composes over whatever
   the trader is already playing.
2. **The take-everything reflex.** Anything useful that comes within touch range is picked
   up, unconditionally. The trader has no judgment about it.
3. **The trade gate.** The trader will not trade away a personal data assistant, and cannot
   put anything except one into a slot.

Everything else is delegation.

## `Load`

**Contract** — base entity load, then two values from the configuration section: starting
health, and a maximum carry weight in kilograms which is scaled by a thousand into the
inventory's internal units. Recomputes total weight.

**Notes** — a disabled line beside it would have given the trader an effectively unlimited
rucksack. The shipped trader is weight-limited like anyone else, which matters because it
is what stops a trader's stock growing without bound as the player sells to it.

## `net_Spawn`

**Contract** — requires a trader spawn record. Spawns the inventory-owner side first (which
creates the personal data assistant), then the entity and script-entity sides. Sets money
from the spawn record. Installs the head-turn callback on the head bone in the custom
callback slot — the slot that runs after animation and physics, so the turn composes on top
of the pose rather than being overwritten. Finally sets the scheduler bounds.

```text
schedule_minimum =  100 milliseconds
schedule_maximum = 2500 milliseconds
```

**Notes** — these bounds tell the scheduler how often the trader may be updated at best and
at worst as load and distance grow. The source carries a comment recording that the two were
once tied to the network latency and that the relation was deliberately broken. A trader is
cheap to update, so the wide band costs nothing; a rebuild should keep the shape — a
per-class minimum and maximum update interval — rather than the numbers.

## `LookAtActor` / `BoneCallback` — the head turn

**Contract** — called from the animation system with the head bone's instance, after the
pose is computed. Rotates the bone about its own pitch axis by an angle derived from the
horizontal angle between the trader's body facing and the direction to the player. Does
nothing beyond twenty metres. The rotation magnitude is the absolute angular difference
clamped to one radian, signed against the direction of the turn.

```text
FUNCTION look_at_player(bone)
  player_pos = current view entity position
  IF distance(player_pos, own position) > 20 metres THEN RETURN

  (wanted_heading, _) = heading_and_pitch(player_pos - own position)
  (body_heading, _, _) = heading_pitch_bank(own transform)

  signed  = normalize_signed(wanted_heading - body_heading)
  # clamp the magnitude so the head never exceeds about 57 degrees of yaw,
  # which is what stops the neck from visibly breaking when the player
  # walks behind a trader who is facing a counter
  amount  = clamp(|signed|, 0, 1 radian)
  IF signed > 0 THEN amount = -amount

  bone.transform = bone.transform composed with a rotation of `amount`
```

**Invariants** — the rotation is applied in the bone's own frame, about the axis the bone
calls pitch. Whether that is the model's yaw depends on the skeleton's convention; for the
shipped human skeleton it is. A rebuild with a different rig must find the equivalent axis
rather than copying the component.

**Notes** — it tracks the *current view entity*, not the actor, so under the debug camera
tool a trader stares at whatever the camera inhabits. And it re-reads the player's position
every frame with no rate limit or smoothing, so the head snaps to the clamp as the player
crosses the boundary rather than easing into it.

## `OnEvent` — take, drop, buy, sell

**Contract** — handles four authoritative events, in two pairs that share their handling.

```text
FUNCTION on_event(packet, type)
  base.on_event(...)  ;  inventory_owner.on_event(...)

  SELECT type
    trade_buy, ownership_take:
      item = find_object(packet.read_id())
      IF inventory can take item
        reparent item to self
        inventory.take(item)
      ELSE
        # tell the authority to give it back, so the server's view of
        # ownership never diverges from the trader's actual inventory
        send ownership_reject naming the item

    trade_sell, ownership_reject:
      item = find_object(packet.read_id())
      just_before_destroy = packet has more AND packet.read_flag()
      # an item sold, or one about to be destroyed, must not get a physics
      # body on the way out — it would fall on the floor and then vanish
      dont_create_shell = (type == trade_sell) OR just_before_destroy
      item.mark_pre_destroy(just_before_destroy)
      inventory.drop(item, just_before_destroy, dont_create_shell)

    transfer_ammo:
      nothing
```

**Invariants** — buying and taking are the same operation to the trader; so are selling and
dropping. The only thing that distinguishes a sale is that the item must not spawn a
physics body. The money side of a trade is settled elsewhere, by the trade machinery.

## `feel_touch_new` — the take-everything reflex

**Contract** — when a new object enters touch range: if the trader is alive, is the
authority, and the object is an inventory item flagged as useful to non-player characters,
send an ownership-take event for it. Unconditional — no room check, no value judgment, no
trade-relevance test.

**Notes** — the trader logs each take and each drop to the engine console unconditionally.
The stalker's equivalent handler wraps the same diagnostics in a compile-time silence
switch; the trader's does not, so a level with several traders and loose items produces a
continuous stream of console lines in a shipping build. A rebuild should not reproduce it.

## `shedule_Update` / `UpdateCL`

**Contract** — the scheduled update runs the base entity update, advances the inventory
owner, and then **either** runs the Lua action queue, if a script has taken control, **or**
thinks. The frame update advances sounds and, when neither a script nor a script animation
is running, advances the dialogue-driven animation.

## `Think`

**Contract** — **empty.** This is the trader's entire autonomous decision-making.

A rebuilder should read it as a statement of architecture, not as an omission: the trader
class deliberately supplies no behaviour, so that all trader behaviour is authorable in Lua
and in dialogue data. Every trader in the shipped games is scripted from the outside.

## `tfGetRelationType`

**Contract** — asks the reputation registry for the relation between this trader and the
other party, provided the other party is an inventory owner and not a creature. A creature
never has a reputation relation with a trader. Falls back to the base entity's rule when the
registry has nothing to say.

## `AllowItemToTrade` / `CanPutInSlot` / `can_attach` / `use_bolts`

**Contract** — the inventory policy, and it is entirely negative. A *dead* trader will trade
anything (its corpse is lootable without restriction). A living trader refuses to trade its
personal data assistant, and otherwise defers to the generic rule. It will put nothing into
a slot except a personal data assistant, attaches nothing, and does not use bolts.

## `g_fireParams` / `g_WeaponBones`

**Contract** — present because the entity contract demands them. The weapon bones resolve to
the standard human hand bones. The fire parameters return the trader's centre and a
*horizontal forward direction with no aim at all* — a trader that somehow fired a weapon
would shoot straight ahead from its chest. Nothing in the shipped game makes a trader fire.

## `ArtefactPrice` / `BuyArtefact`

**Contract** — pricing returns the artefact's own cost, ignoring any order list. Purchase
**always refuses**.

These are the remains of a generated-quest feature: a trader was to hold a list of artefacts
it wanted, price them above market and strike them off the list when bought. The list is
gone; the two functions are the stubs it left. A rebuilder should not try to infer the
feature from them — take the names as evidence that such a feature existed, and nothing more.

## `OnStartTrade` / `OnStopTrade`

**Contract** — set and clear the busy flag and fire the corresponding script callback. The
busy flag is what lets other systems — and scripts — know a trade screen is open on this
trader.

## `net_Export` / `net_Import`

**Contract** — **asymmetric, and broken.** Export writes nothing at all; its two writes are
commented out. Import reads a float and then a money value. A trader replicated over the
network would read whatever followed in the stream as its wealth. The shipped build has no
working transport, so the path is unreachable. See
[Seam: Networking transport](../../../../SYSTEM-REQUIREMENTS.md#seam-networking-transport).
A rebuild implementing multiplayer must write this pair afresh, not port it.

## `save` / `load` / `spawn_supplies` / `reinit` / `reload` / `_construct` / `net_Destroy`

**Contract** — each chains the corresponding operation across the mixins in a fixed order.
The order is load-bearing for construction and re-initialisation: the script-entity side is
reset before the living-entity side, which is before the inventory-owner side, and the sound
and animation components last — because the animation component resolves a bone handle from
the visual, which the entity side must already have installed.
