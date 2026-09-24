# src/xrGame/script_game_object_inventory_owner.cpp

> Information portions, dialogue and the talk screen, inventory movement and money, the whole relations model, character identity, tasks, restrictors, doors, weapon attachments, item upgrades and the actor's weight and speed limits.

**Needs** — [`script_game_object.h`](script_game_object.h.md) · [`script_game_object_impl.h`](script_game_object_impl.h.md) · [`InventoryOwner.h`](InventoryOwner.h.md) · [`Inventory.h`](Inventory.h.md) · [`InventoryBox.h`](InventoryBox.h.md) · [`InfoPortion.h`](InfoPortion.h.md) · [`relation_registry.h`](relation_registry.h.md) · [`character_info.h`](../xrServerEntities/character_info.h.md) · [`GameTask.h`](GameTask.h.md) · [`GameTaskManager.h`](GameTaskManager.h.md) · [`Actor.h`](Actor.h.md) · [`CustomOutfit.h`](CustomOutfit.h.md) · [`WeaponMagazined.h`](WeaponMagazined.h.md) · [`attachable_item.h`](attachable_item.h.md) · [`space_restriction_manager.h`](space_restriction_manager.h.md) · [`doors_manager.h`](doors_manager.h.md) · [`smart_cover_object.h`](smart_cover_object.h.md) · [`ui/UITalkWnd.h`](ui/UITalkWnd.h.md) · [`UIGameSP.h`](UIGameSP.h.md) · [`xrMessages.h`](../xrServerEntities/xrMessages.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: guarded delegation, plus several network-event constructions

## Purpose

The largest of the nine files the [game object facade](script_game_object.h.md) is split
across, and the one the quest scripts live in. Its subject is the **inventory owner** —
anything that can carry, trade, talk and hold an opinion of others — plus a long tail of
actor-specific tuning that accreted here.

Three things in it are not guarded delegation and carry the file: **information portions**,
which are how the games track what the player knows; the **relations model**, which is how
the world decides who shoots whom; and the several operations that move items and money by
raising **network events** rather than by calling directly.

## State

`Stateless.` The facade's door registration is declared in
[`script_game_object.h`](script_game_object.h.md) and used here.

## Information portions

**Contract** — `give_info_portion(id)` and `disable_info_portion(id)` set and clear a named
flag on an inventory owner, answering whether the object could hold one at all. `has_info`,
`dont_has_info` and `info_time(id)` read them back, the last returning when the portion was
granted.

**Invariants**

- An information portion is a **globally named boolean per character**, not a payload.
  Granting one fires the dialogue and task systems' conditions, which is why granting is a
  separate operation from anything the player does — the entire quest state of the games is
  a set of these.
- The grant time is recorded and readable. Quests use it to sequence: "this became true
  before that did".
- `dont_has_info` exists alongside `has_info` because the authored condition tables in the
  dialogue data test one or the other by name, and negation is not available there.

## Inventory movement and money

**Contract** — `drop_item`, `drop_item_and_teleport`, `make_item_active`, `transfer_item`,
`transfer_money`, `give_money`, `money`, `unload_magazine`, plus iteration.

**Invariants**

- **Moving an item or money between owners is done by raising engine events, not by
  calling.** A transfer is spelled as a *sell* event addressed to the giver followed by a
  *buy* event addressed to the receiver; activating an item is spelled as up to three
  events — move whatever occupies the slot back to the pack, move this item into the slot,
  activate the slot — in that order. This is the same reasoning as the scripted hit in
  [`script_game_object2.cpp`](script_game_object2.cpp.md#hit--applying-a-script-built-hit):
  inventory is authoritative state that must travel one path.
- The three activation events must be sent **in that order and as three events**, because
  each one's handler assumes the previous has landed. A rebuild that fuses them has to
  preserve the intermediate states, since the displaced item must be in the pack before the
  new one enters the slot.
- `transfer_money` checks the giver has enough and reports a script error if not, rather
  than going negative. `give_money` creates money from nothing and has no such check — it is
  the reward path.
- `mark_item_dropped` / `marked_dropped` flag an item as *deliberately* dropped, which stops
  the owner's own logic picking it straight back up.

## Iteration over contents

**Contract** — three iterators with three different contracts, and the differences are
real:

```text
for_each_inventory_item(function)
  # visits every *available* item; the function's answer is ignored
  # the function is called with (item, this owner)

iterate_inventory(function, bound_object)
  # visits every item the owner holds, including equipped ones
  # STOPS as soon as the function answers true
  # the function is called with (bound_object, item)

iterate_inventory_box(function, bound_object)
  # the same, over a container's contents, looked up by identifier
  # STOPS on a true answer
```

**Invariants**

- The stopping ones stop on **true**, which reads as "found it". A script that returns a
  value by accident from the last statement of its function silently visits one item.
- The argument order differs between the non-stopping form and the stopping ones — owner
  last in one, bound object first in the others. Both are frozen; a rebuild keeps both.
- The container iterator resolves each identifier to a live object and **skips what it
  cannot find**. A container may reference an item that is not currently instantiated, so
  the skip is correct, not defensive.

## The relations model

**Contract** — the model has three independent layers, and conflating them is the most
common mistake against this surface:

```text
goodwill      : a number from one character to one other character
                get_goodwill, set_goodwill, force_set_goodwill, change_goodwill (delta)
community     : the faction a character belongs to
                character_community, set_character_community(name, squad, group)
                community_goodwill(community), set_community_goodwill(community, value)
relation      : the resolved answer — friend, neutral, enemy
                set_relation(kind, other)
attitude      : the effective number, combining personal goodwill with community standing
                get_attitude(other)
sympathy      : how strongly this character's opinion spreads to its community
                get_sympathy, set_sympathy
reputation    : standing with the world at large
                character_reputation, set_character_reputation, change_character_reputation
rank          : combat standing
                character_rank, set_character_rank, change_character_rank
```

**Invariants**

- **Goodwill is directed and personal; attitude is what the game actually uses.** Attitude
  folds a character's personal goodwill together with the standing between their
  communities, so shooting one member of a faction turns the whole faction against the
  player without any per-character bookkeeping. A rebuild that stores only the resolved
  relation cannot reproduce that.
- `set_goodwill` respects the engine's own arbitration and `force_set_goodwill` does not.
  Both ship and both are used.
- **Sympathy is the coupling constant** between personal and community: it decides how much
  of a change to one character's opinion propagates outward. It is per-character, so a
  faction leader's opinion carries further than a rank-and-file member's.
- Setting a community **also reassigns the character's team, squad and group**, in one call,
  because a faction change that left the combat grouping behind would leave a character
  shooting its new allies. The community name is validated and the whole operation refused
  if it does not resolve.

## Character identity

**Contract** — `profile_name`, `character_name`, `character_icon`, `set_character_icon`
and `sound_voice_prefix` read and write who a character appears to be: the authored profile
it was built from, its display name, its portrait, and the stem of the sound set it speaks
with.

## Talking

**Contract** — `is_talking`, `stop_talk`, `enable_talk`, `disable_talk`, `is_talk_enabled`,
`run_talk_dialog(partner, disable_break)`, `allow_break_talk_dialog(allowed)`,
`switch_to_trade`, `switch_to_upgrade`, plus `enable_trade` / `disable_trade` /
`is_trade_enabled` and the same trio for the upgrade screen, and
`add_iconed_talk_message(...)` in two shapes.

**Invariants**

- Talking, trading and upgrading are **three separately gated capabilities** on one
  character. A trader can be barred from trading while still talking, which is how a quest
  makes someone refuse business.
- `run_talk_dialog` is addressed to the **actor**, naming the partner — not the other way
  round — because the conversation is a player-facing screen and belongs to the player's
  session. Calling it on anything but the actor is a script error.
- `switch_to_trade` and `switch_to_upgrade` only act **while the talk screen is already
  shown**, and silently do nothing otherwise. They are transitions within a conversation,
  not ways to open one.
- `switch_to_talk` exists and **always fails**, deliberately: the method is a placeholder
  that was never implemented and aborts with a message if called. A rebuild should either
  implement it or not export it.
- `allow_break_talk_dialog` decides whether the player may walk out mid-conversation. Quest
  scripts disable it for conversations that must be seen through.
- The iconed talk message is drawn into the conversation as an extra line with an image,
  used for handing over documents and photographs. It does nothing when the talk screen is
  not up, and it falls back to a default layout template when none is named.

## Tasks and news

**Contract** — `give_task(task, delay, check_existing, timer)`, `get_task(id, only_active)`,
`set_active_task`, `is_active_task`, `game_task_state(task, objective)` and
`set_game_task_state(state, task, objective)` manage the player's quest log;
`give_game_news(...)` and `clear_game_news` post to the news ticker.

**Invariants** — a task has **objectives** addressed by index, each with its own state, and
the task's own state is derived from them. Scripts drive objectives, not tasks, for
anything with more than one step.

## Restrictors

**Contract** — `add_restrictions(out, in)`, `remove_restrictions(out, in)`,
`remove_all_restrictions`, the four readers (`in_restrictions`, `out_restrictions`,
`base_in_restrictions`, `base_out_restrictions`) and three queries:

```text
accessible_position(p)          -> may this entity be at p
accessible_vertex_id(v)         -> may it be at navigation vertex v
accessible_nearest(p, out_p)    -> the nearest place it MAY be, to p
```

**Invariants**

- Restrictors are named as **space-separated lists of object names in one text value**, both
  going in and coming back. Names that do not resolve are skipped silently, so a typo
  removes a restriction rather than reporting one.
- *In* restrictors are permitted regions and *out* restrictors are forbidden ones, and an
  entity's effective space is the intersection. The **base** readers give the set from the
  entity's authored configuration, the plain readers the current set — the difference is
  what a script added, which is what lets a script restore rather than remember.
- `accessible_nearest` **reports a script error when the position is already accessible**
  and answers the invalid marker. That is not a failure: it is telling the script it asked
  the wrong question, because the answer would be the position itself and the script
  probably meant to test accessibility first.
- `accessible_vertex_id` answers false for a vertex that is not on the level graph at all,
  rather than failing — the distinction between "not allowed there" and "there is no there"
  is collapsed deliberately, since both mean the script must choose elsewhere.

## Doors

**Contract** — `register_door` and `unregister_door` add and remove this physics object from
the navigation layer's door table; `on_door_is_open` and `on_door_is_closed` notify it of
state changes; `is_door_locked_for_npc`, `lock_door_for_npc`, `unlock_door_for_npc` and
`is_door_blocked_by_npc` read and write whether creatures may path through.

**Invariants**

- **Registration is the facade's one owned resource** and is released in the facade's
  destructor — see
  [`script_game_object_use.cpp`](script_game_object_use.cpp.md#facade-lifetime). Every
  other method here asserts the object is registered.
- A script must tell the door table when the door opens or closes; the table does not
  observe the physics object. That is deliberate — a door's *navigational* state is not its
  geometric one, and a door swinging in a blast should not re-plan every creature's path.
- **Locked** and **blocked** are different: locked is a policy a script sets, blocked is the
  table reporting that a creature is standing in the doorway.

## Weapon attachments and item upgrades

**Contract** — `attach_addon(item)` and `detach_addon(section)` fit and remove a scope,
silencer or grenade launcher, each checking the weapon's own compatibility rule first and
doing nothing when it refuses. Six readers report which addons a weapon can take and which
are fitted. `add_upgrade`, `install_upgrade`, `can_add_upgrade`, `can_install_upgrade`,
`has_upgrade`, `has_upgrade_group`, `has_upgrade_group_by_upgrade_id` and
`iterate_installed_upgrades` cover the item upgrade tree.

**Invariants** — *add* and *install* are distinct: adding records the upgrade as owned,
installing applies its effect. The four-way split between "can add", "add", "can install"
and "install" is what lets the upgrade screen show an option as available, affordable or
already taken.

## Actor limits and movement tuning

**Contract** — read/write pairs for the actor's maximum carry weight, maximum walking
weight, jump speed, sprint coefficient, run coefficient and backward-run coefficient; the
additional weight allowances granted by an outfit or a backpack; total carried weight; an
item's own weight; plus `allow_sprint`, `enable_movement`, `movement_enabled`,
`restore_weapon`, `hide_weapon`, `active_slot`, `activate_slot`, `item_in_slot`,
`active_detector`, `animation_slot`, `belt_size`, `item_on_belt`, `is_on_belt` and
`actor_look_at_point`.

**Invariants**

- Maximum weight and maximum **walking** weight are different limits: exceeding the first
  stops the player moving at all, the second only stops them walking normally. Both are
  raised by outfits and backpacks through the *additional* allowances, which is why those
  are read from either kind of item.

**Notes**

The additional-weight accessors are **wrong in the original**: the walking-allowance reader
returns the carry allowance's field, so the two readers answer the same number while the
two writers write different fields. A script that sets the walking allowance cannot read it
back. A rebuild should give each its own field on both sides.

## Stalker combat tuning

**Contract** — grenade throwing (`can_throw_grenades`, `throw_time_interval`,
`group_throw_time_interval`), per-weapon `aim_time`, `special_danger_move`,
`sniper_update_rate`, `sniper_fire_mode`, `aim_bone_id`, `register_in_combat` /
`unregister_in_combat`, `find_best_cover(threat_position)`, `take_items_enabled`,
`death_sound_enabled` and `set_play_shell_and_reload_sounds`.

**Invariants** — the group throw interval is separate from the individual one so that a
squad does not throw a volley of grenades on the same frame; the same anti-unison reasoning
as the sound and dwell-time windows elsewhere in the facade. `aim_bone_id` names which bone
of a target a creature aims at, which is how a scripted encounter makes an enemy aim for
the legs.

## `suitable_smart_cover`

**Contract** — asks whether this stalker could use a given smart cover, which is the one
predicate here with an algorithm:

```text
FUNCTION suitable_smart_cover(cover_object) -> bool
  IF cover_object is nothing THEN log script error; RETURN false
  IF this is not a stalker THEN log script error; RETURN false
  IF cover_object is not a smart cover THEN log script error; RETURN false
  IF the cover cannot be fired from THEN RETURN true      # anyone may hide in it
  active = the stalker's currently held item
  IF active is present THEN RETURN active occupies the primary weapon slot
  best = the stalker's best weapon
  IF best is nothing THEN RETURN false                    # unarmed: a firing cover is useless
  RETURN best occupies the primary weapon slot
```

**Invariants** — a cover that can be fired from is suitable **only for a creature with a
primary weapon**. A pistol or a knife does not qualify, because the cover's authored
animations are for a rifle and would not line up. This is the clearest case in the facade
of a behavioural rule that exists because of the *animation data*, and a rebuild with
different animations may need a different rule — but with the shipped data, this one.
