# src/xrGame/xrServer_CL_connect.cpp

> Admits one client: the ordered gauntlet of checks it must pass, and then the replay of the entire world into its lap.

**Needs** — [`xrServer.h`](xrServer.h.md) · [`xrMessages.h`](../xrServerEntities/xrMessages.h.md) · [`xrServer_Objects.h`](../xrServerEntities/xrServer_Objects.h.md) · [`xrServer_Objects_ALife_Monsters.h`](../xrServerEntities/xrServer_Objects_ALife_Monsters.h.md) · [`Level.h`](Level.h.md) · [`xrNetServer/NET_Messages.h`](../xrNetServer/NET_Messages.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: re-serializes every entity into wire packets and mutates spawn flags in place

## Purpose

Two things happen when a client joins, and both are here. First it must prove it may be here —
a chain of four checks, each of which suspends the join and resumes it from a reply. Then the
world must be *replayed* into it: every entity that exists, in an order the client can
reconstruct.

## State

```text
conn_spawned_ids : list<int (16-bit)>   # entities already sent during THIS client's join
```

**Invariants** — the list is cleared at the start of each client's replay and is what makes the
recursive parent-first walk idempotent.

## The admission chain

**Contract** — four gates in a fixed order, each either passing through synchronously or
suspending until a reply arrives. The source warns that changing the order requires changing
the challenge handler to match.

```text
1. copy-protection key      → Check_GameSpy_CDKey_Success
2. game-data authenticity   → NeedToCheckClient_BuildVersion / OnBuildVersionRespond
3. server access / password → Check_ServerAccess, inside the version respond
4. ban list                 → ProcessClientDigest, in xrServer_Connect.cpp
                            → Check_BuildVersion_Success → admitted
```

**Notes** — **the order is by cost and by blast radius.** The cheapest-to-refuse and most
externally-verified check runs first; the ban check runs last because it needs the digest that
the earlier steps established. A rebuild reordering them must re-derive which checks depend on
which identity being known.

Each gate is written as a *predicate that may send a message*, with a named success continuation.
That is an asynchronous state machine written without one, and it works only because each gate
has exactly one outstanding request. A rebuild should make the state explicit, because the
original's version silently does the wrong thing if two replies arrive out of order.

## `Check_GameSpy_CDKey_Success`

**Contract** — the copy-protection gate has passed. Moves to the game-data authenticity gate;
if that gate does not need to run, moves straight on to requesting the client's digest.

## `NeedToCheckClient_BuildVersion`

**Contract** — begin the game-data authenticity gate. Establishes the shared secret, clears the
client's verified flag, and sends a challenge. Answers whether the caller should wait. Answers
no immediately — skipping the gate — when the authenticity check is disabled by a server switch.

## `OnBuildVersionRespond`

**Contract** — the client's answer to the authenticity challenge. Compares the client's reported
digest of its mounted game data against the server's own.

```text
FUNCTION on_build_version_respond(client, packet)
  ours  := our data-authenticity value
  theirs := packet.read_int(64-bit)

  IF ours != theirs AND the mismatch is not being ignored
    refuse(client, data verification failed)
    RETURN

  IF client is the host's own
    proceed to the digest request
  ELSE IF server access check passes
    proceed to the digest request
  ELSE
    refuse(client, password verification failed)
```

**Invariants** — the value is a **64-bit digest over the mounted archives**, computed at connect
time by the file system, over a set of paths the game layer nominates. It is not a version
number: two clients with the same build and different edited data files disagree here. That is
its whole purpose.

**Notes** — two server switches can each disable a gate: one skips the challenge entirely, the
other accepts a mismatch. Both exist for development and both are footguns on a public server. A
rebuild should keep them and make them loud.

The host's own client bypasses the access check but not the authenticity check, which is the
right way round: the host cannot be locked out of its own server, but it is not exempt from
having consistent data.

## `Check_BuildVersion_Success`

**Contract** — the last gate has passed. Marks the client verified and sends it a success
result.

## `SendConnectResult`

**Contract** — tell a client the outcome of its join. Carries a success flag, a refusal-reason
code, a human-readable string, the client's own identifier as the server sees it, **whether this
client is the host**, and the session's option string.

**Disconnects the client when the outcome is failure** — after flushing the outbound buffers, so
the refusal actually reaches it before the connection drops. That flush is the load-bearing
detail: without it the client learns only that it was dropped, not why.

**Notes** — sending the client its own identifier is how a client learns its identity; it has no
other way to know which of the entities it is about to receive is *its*. Sending the session
option string here rather than earlier means the client configures itself from the same string
the server parsed, so the two cannot disagree.

Refusal carries both a machine code and a string, because the code selects the client's behaviour
— retry, prompt for a password, give up — and the string is what the player reads.

## `SendProfileCreationError`

**Contract** — refuse a join that failed for account reasons. Same shape as a refusal result, with
a fixed reason code. **Does not disconnect the host's own client**, which would take the server
down with it.

## `SendConnectionData`

**Contract** — replay the world. Clears the per-join sent list, marks every entity unprocessed,
then walks every entity through the parent-first spawn sender. Finishes by starting the upload of
the server's description, logo and rules.

**Invariants** — the two passes are separate: *all* entities are marked unprocessed before *any*
is sent, because the recursive walk consults the flag across the whole table.

## `Perform_connect_spawn`

**Contract** — send one entity's spawn record to a joining client, **having first sent its
parent's**. Skips an entity already sent during this join, one already processed, and any phantom.
Recurses to the parent before writing itself.

```text
FUNCTION perform_connect_spawn(entity, client, packet)
  packet.reset()
  IF entity.id IS IN conn_spawned_ids THEN RETURN      # already sent this join
  conn_spawned_ids.append(entity.id)

  IF entity.net_processed THEN RETURN
  IF entity is a phantom THEN RETURN

  parent := entities[entity.parent_id]
  IF parent EXISTS THEN perform_connect_spawn(parent, client, packet)   # parent first

  saved_flags := entity.spawn_flags
  entity.spawn_flags.set(SPAWN_CARRIES_UPDATE)

  IF entity has no owning client
    IF entity is flagged "spawn as the player"
      client.owner := entity
      entity.display_name := client.player_state.name
    entity.owner := client
    entity.write_spawn(packet, with_client_data: true)
    entity.write_update(packet)
    IF NOT entity.keep_saved_data_anyway
      entity.client_data.clear()              # see Notes
  ELSE
    entity.write_spawn(packet, with_client_data: false)
    entity.write_update(packet)

  entity.spawn_flags := saved_flags
  send reliably to client
  entity.net_processed := true
```

**Invariants** — **a client never receives a child before its parent.** An entity's spawn record
names its parent by identifier, and the client must have that parent to attach to. The recursion
guarantees it regardless of the table's iteration order, which is the same undefined order the
load-order warning in [`xrServer.h`](xrServer.h.md) is about — here it is handled properly.

The spawn flag is set, the record written, and the flag restored, so the mutation is invisible
outside the call. The flag tells the receiver that an update record follows the spawn record in
the same packet.

**Notes** — **The client-data blob is sent exactly once and then destroyed.** It is the
client-side state an entity was saved with — the part only a live client object knows how to
interpret — and it exists on the server only to be handed to the first client that takes ownership.
Entities that opt out keep it, which is how an entity that may be handed to a *second* client later
survives. A rebuild must reproduce the single-delivery rule or a re-joining client will restore
stale state.

**Ownership is assigned here, as a side effect of replay.** An entity with no owner is given to the
joining client. In a session with several clients that is a race decided by join order, and it is
the other half of the abandoned migration story in
[`xrServer_balance.cpp`](xrServer_balance.cpp.md).

Naming the player's entity after the client's player state is what makes a corpse in the world
carry a player's name.

## `OnCL_Connected`

**Contract** — the client is admitted. Marks it accepted — which is what makes broadcasts start
reaching it — exports the game type and the full game state to it, replays the world, and tells
the rules layer a player has connected.

**Invariants** — the accepted flag is set **first**, before the replay, so that anything
broadcast during the replay reaches the client too. The player state must already exist by this
point; the source treats its absence as a message-sequence error and bails rather than
continuing.

## `SendConfigFinished`

**Contract** — a single bare message telling a client the configuration phase is over. The client
uses it as the signal to leave its loading screen.
