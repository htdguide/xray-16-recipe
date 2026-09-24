# src/xrGame/inventory_owner_info.cpp

> What a character *knows*: the story-information portions they have received, how one is granted or withdrawn, and the script actions a grant fires.

**Needs** — [`InventoryOwner.h`](InventoryOwner.h.md) · [`InfoPortion.h`](InfoPortion.h.md) · [`GameObject.h`](GameObject.h.md) · [`Level.h`](Level.h.md) · [`alife_simulator.h`](alife_simulator.h.md) · [`alife_registry_container.h`](alife_registry_container.h.md) · [`alife_registry_wrappers.h`](alife_registry_wrappers.h.md) · [`script_game_object.h`](script_game_object.h.md) · [`game_object_space.h`](game_object_space.h.md) · [`ai_space.h`](ai_space.h.md) · [`xrServerEntities/xrMessages.h`](../xrServerEntities/xrMessages.h.md) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: a per-character set keyed by name, with a script callback on change

## Purpose

An **info portion** is one named fact a character may know. It is the game's entire story
state: quest progress, dialogue unlocks, faction relations, everything a script wants to
branch on. Scripts and dialogue nodes grant and revoke them, and ask whether a character has
one.

This file is the *owner's* half of that system — the set each character carries, and the
grant and revoke paths. The portion's own authored content — the script functions it runs,
the portions it cancels — is in [`InfoPortion.h`](InfoPortion.h.md).

The design decision worth holding: this is **not a global flag table**. Every character has
their own set, and "the player knows X" is just the player's set containing X. That is what
lets one dialogue branch on what the speaker knows independently of what the listener knows.

## State

```text
RECORD InfoData
  info_id      : text        # the portion's name; interned
  receive_time : int         # the in-game clock at the moment it was granted

RECORD KnownInfoSet          # per character, held in the alife registry
  entries : list<InfoData>   # insertion order preserved; distinct by info_id
```

**Invariants**
- Distinct by name. A second grant of a portion already held is a no-op and is *reported as
  such* — the return value is how callers avoid firing the same story beat twice.
- Stored in the **alife registry**, not on the client object. The set survives the
  character going offline and back, and it is what the save file persists. A character who
  has never been online still accumulates portions.
- The receive time is recorded and never read by the engine. Scripts read it.

## `TransferInfo`

**Contract** — the only entry point that should be used to change a character's knowledge.
Takes a portion name and a direction. Emits an authoritative event so the server record
learns of the change, **and** applies the change locally in the same call. Not idempotent
in effect: the grant path runs the portion's script actions.

```text
FUNCTION TransferInfo(info_id, adding)
  emit event INFO_TRANSFER to self, carrying (self.id, info_id, adding)
  IF adding THEN OnReceiveInfo(info_id) ELSE OnDisableInfo(info_id)
```

**Notes** — the local application is deliberately *not* deferred until the event comes back.
Story scripts run in long chains — granting a portion whose actions grant three more — and a
chain that had to wait a frame for each link would interleave with everything else in the
world. The cost is that the event arrives at the receiver later and finds the change already
made, which is harmless because both paths converge on the same set.

The event is addressed to the character themselves, not broadcast. Other characters learn
nothing; there is no gossip mechanism at this layer.

## `OnEvent`

**Contract** — handles the one event this mix-in owns: an information transfer. Unpacks
sender, portion name and direction and routes to the grant or revoke path. The sender
identifier is read off the wire and not used — the portion always lands on the receiver.

**Notes** — the sender field is written by every producer and consumed by none. It looks
like the start of an unfinished "who told me this" mechanic. Unrecovered: nothing in the
tree reads it.

## `OnReceiveInfo`

**Contract** — grants a portion. Reports `false` and does nothing further if the character
already holds it. Otherwise records it with the current in-game time, runs the portion's
authored script actions, and then revokes every portion the new one declares obsolete.
Recursive through that last step.

```text
FUNCTION OnReceiveInfo(info_id) -> bool
  IF known set already contains info_id
    RETURN false                       # the caller's signal not to re-fire a story beat
  known set.append(InfoData{ info_id, current in-game time })

  portion = load authored portion info_id
  portion.run_script_actions(self)
  FOR EACH obsolete IN portion.disable_infos
    TransferInfo(obsolete, false)
  RETURN true
```

**Invariants** — the portion is recorded **before** its script actions run. Those actions
routinely grant further portions and ask whether this one is held; recording afterwards
would let a portion's own actions re-enter and grant it twice.

**Notes** — the authored portion is loaded from configuration on *every* grant rather than
being resolved once. Portions are small and grants are rare, so this costs nothing in
practice; a rebuild is free to resolve them at startup.

The cancellation list is what keeps the set bounded: a quest's "go here" portion is
cancelled by its "you arrived" portion, so a long game does not accumulate every beat it
ever passed. Cancellation is one level deep per grant but recursive overall, since each
cancelled portion is withdrawn through the full path.

This method is declared const and mutates the set. The set lives behind the alife registry
rather than in the object, so the language's const rule never noticed — an artifact, not a
decision.

## `OnDisableInfo`

**Contract** — revokes a portion. Silently does nothing if it is not held. Runs no script
actions: **revocation is not symmetric with grant**. A portion's authored content describes
what happens when it is learned, and there is no "unlearn" hook anywhere in the system. A
rebuild that adds one changes the dialogue data's meaning.

## `HasInfo` / `GetInfo`

**Contract** — the two queries. `HasInfo` is the predicate scripts and dialogue
preconditions call constantly. `GetInfo` additionally yields the record, which is how a
script reads when a portion was granted. Both tolerate a character whose registry entry has
never been created, reporting "not held" rather than failing — a character who has learned
nothing has no storage at all.

**Notes** — both are linear scans of the character's list. The list is short for everyone
except the player, whose list runs to a few hundred entries late in a game, and the predicate
is called from every dialogue precondition on every line. This is the one place in the
system where the data structure is visibly the wrong one; a rebuild should use a set keyed by
the interned name and keep insertion order separately if the order matters to scripts.

## `DumpInfo`

**Contract** — development-build listing of everything a character knows. A diagnostic.
