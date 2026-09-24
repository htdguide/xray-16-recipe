# src/xrGame/game_sv_mp_script.cpp

> Makes the multiplayer server rules subclassable from Lua, and hands scripts the one thing they cannot otherwise reach: the damage numbers inside a hit packet.

**Needs** — [`game_sv_mp_script.h`](game_sv_mp_script.h.md) · [`game_sv_mp.h`](game_sv_mp.h.md) · [`xrServer.h`](xrServer.h.md) · [`Level.h`](Level.h.md) · [`ai_space.h`](ai_space.h.md) · [`xrServerEntities/xrServer_Objects_ALife_Monsters.h`](../xrServerEntities/xrServer_Objects_ALife_Monsters.h.md) · [`xrServerEntities/xrServer_script_macroses.h`](../xrServerEntities/xrServer_script_macroses.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: rewrites floats at fixed byte offsets inside a packet

## Purpose

The server half of "a game mode written in Lua": the class a script mode derives from, the
overridable hooks, and a respawn implementation exposed for scripts to call directly. It
also contains three functions that reach into a hit packet at hard-coded byte offsets, which
is the most fragile code in this group and the clearest example of a decision a rebuild must
make differently.

## State

`Stateless.` The mode's state is its base's.

## `SetHitParams` / `GetHitParamsPower` / `GetHitParamsImpulse`

**Contract** — read or overwrite the damage and the impulse of a hit event, **in place**, at
fixed byte offsets within the packet: damage at offset 16, impulse at offset 34. Each saves
the packet's cursor, moves it, reads or writes one float, and restores it.

**Invariants** — this is how a script mode changes the damage of a shot it did not construct:
the hit arrives from a client as an opaque packet, the mode wants to scale it, and the only
way in is to know where the numbers are.

The offsets are the **hit event's wire layout**, expressed nowhere else. They are a duplicate
of the format defined by the hit event's own serialization, and nothing checks the two agree.
A rebuild must not copy the numbers; it must parse the hit event into a record, let the mode
modify the record, and re-serialize — which is what having a typed event would have given for
free.

The cursor save and restore means the calls are transparent to a caller mid-stream, which
they must be: the mode inspects the hit while the packet is being read for other purposes.

**Notes** — the write path saves and restores the *write* cursor while the read paths save
and restore the *read* cursor. Both are needed because the packet has two independent
positions, and a rebuild with one cursor per direction must keep both.

## `SpawnPlayer`

**Contract** — respawns a client as an actor or as a spectator, replacing whatever body it
had. Exported to script, so a script mode controls its own respawn policy.

```text
FUNCTION spawn_player(client, section, skin, respawn_point)
  record = the client's player record
  record.set_flag(permanently_dead)        # see the invariant

  IF the client owns a body THEN
    IF it is an actor THEN
      permit its corpse to be removed, and remember the corpse
    ELSE IF it is a spectator THEN
      reassign it to the server's own client, then destroy it

  entity = begin a spawn of `section`
  entity.name  = the client's player name
  entity.flags = local, and spawned as a player

  FAIL WITH not_an_actor_or_spectator UNLESS the entity is one of the two

  IF actor THEN
    entity.team   = the record's team
    entity.visual = skin, when a skin was named
    record.reset_flag(permanently_dead)
    record.respawn_time = now
    place it at the respawn point's position and orientation
  ELSE                                     # spectator
    IF the client's previous actor still has a position THEN place it there
    ELSE assign it a respawn point

  finish the spawn, owned by this client
  record.game_id = the new body's identifier    # pushes the old one into the history
  signal that clients must resynchronize
```

**Invariants**

- The permanently-dead flag is **set at the start and cleared only on the actor path**. A
  client respawning as a spectator therefore stays marked permanently dead, which is exactly
  right: the flag means "has no living body", and it is what suppresses the quick-speech
  radio and the input gate for a spectator.
- An outgoing actor's corpse is kept and scheduled for removal separately — the body must
  persist a moment so other clients see it fall.
- An outgoing spectator is **reassigned to the server's own client before being destroyed**.
  Destroying an entity a client still owns would leave the client owning nothing; handing it
  to the server first keeps the ownership invariant true at every instant.
- A spectator inherits the position of the body it replaced, so the camera does not jump on
  death. Only when there is no such body does it take a respawn point.
- Adopting the new body identifier goes through the record's own setter, which pushes the old
  one into the recent-bodies history — so a hit report already in flight against the old body
  still resolves to this player.

**Notes** — the ownership link is re-established after the spawn completes, and only if the
client has an owner by then. The spawn machinery sets it; this re-assertion is defensive.

## the overridable surface

**Contract** — nine methods a script may override, each registered with both the compiled
implementation and a base call: the mode's name, the periodic update, the event handler, the
option-string parse, the state export, the two round boundaries, the player-versus-player hit
hook, and the player-record constructor.

Four of these are declared here as plain delegations to the base. They exist **only** so the
binding layer has a virtual it can intercept: without an override point the compiled base
would be called directly and a script override would never run. A rebuild whose dispatch
crosses the language boundary naturally does not need them.

**Invariants** — the player-record constructor transfers ownership to the engine; see the
note in [`game_cl_mp_script.cpp`](game_cl_mp_script.cpp.md), which carries the same
unresolved asymmetry.

## `Create`

**Contract** — parses the match option string: runs the base's parse, then hands the same
string to the script-overridable single-argument form, so a script mode reads its own options
after the engine has read the shared ones.

## `game_sv_mp::script_register`

**Contract** — registers the non-derivable multiplayer server base with three members: kill a
player, send the kill announcement, and force a resynchronization. A script mode that does
not want to subclass can still drive these.

**Notes** — a weapon-spawning helper is registered out. Script modes equip players through the
buy system instead.

## `game_sv_mp_script::script_register`

**Contract** — registers the derivable class: a constructor, the nine overridables, and six
direct calls — team data, the respawn above, the phase switch, and the three hit-packet
accessors.

**Notes** — a delayed round-end hook is registered out along with its wrapper, so a script
mode can react to a round ending but not to the delay before it. The one-argument enumeration
it would have taken is the reason: the wrapper machinery of the original could not carry it.
