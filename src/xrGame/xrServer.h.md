# src/xrGame/xrServer.h

> Declares the authoritative side of the world: the entity table, the client table, the identifier allocator, and every message the server answers.

**Needs** — [`xrServer.cpp`](xrServer.cpp.md) · [`xrNetServer/NET_Server.h`](../xrNetServer/NET_Server.h.md) · [`game_sv_base.h`](game_sv_base.h.md) · [`id_generator.h`](id_generator.h.md) · [`secure_messaging.h`](secure_messaging.h.md) · [`xrServer_updates_compressor.h`](xrServer_updates_compressor.h.md) · [`xrClientsPool.h`](xrClientsPool.h.md) · [`xrEngine/mp_logging.h`](../xrEngine/mp_logging.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — [`Level.h`](Level.h.md) · [`Level_SLS_Default.cpp`](Level_SLS_Default.cpp.md) · [`Level_SLS_Save.cpp`](Level_SLS_Save.cpp.md) · [`Level_network.cpp`](Level_network.cpp.md) · [`Level_network_Demo.cpp`](Level_network_Demo.cpp.md) · [`Level_network_messages.cpp`](Level_network_messages.cpp.md) · [`Level_network_start_client.cpp`](Level_network_start_client.cpp.md) · [`Level_start.cpp`](Level_start.cpp.md) · [`PDA.cpp`](PDA.cpp.md) · [`alife_dynamic_object.cpp`](alife_dynamic_object.cpp.md) · [`alife_monster_base.cpp`](alife_monster_base.cpp.md) · [`alife_simulator_base.cpp`](alife_simulator_base.cpp.md) · [`alife_simulator_script.cpp`](alife_simulator_script.cpp.md) · [`alife_storage_manager.cpp`](alife_storage_manager.cpp.md) · _and 48 more_
**Tier floor** — T1: a fixed-width entity identifier packed into wire messages, and packet buffers copied by raw size

## Purpose

Declares the surface implemented across [`xrServer.cpp`](xrServer.cpp.md) and roughly
twenty sibling files, each carrying one operation or one message family. The split into
files is by *subject*, and the twins mirror it:

| file | subject |
|---|---|
| [`xrServer.cpp`](xrServer.cpp.md) | the update loop, the message switch, the client table |
| [`xrServer_Connect.cpp`](xrServer_Connect.cpp.md) / [`xrServer_Disconnect.cpp`](xrServer_Disconnect.cpp.md) | bringing the server up and down |
| [`xrServer_CL_connect.cpp`](xrServer_CL_connect.cpp.md) / [`xrServer_CL_disconnect.cpp`](xrServer_CL_disconnect.cpp.md) | one client arriving and leaving |
| [`xrServer_process_spawn.cpp`](xrServer_process_spawn.cpp.md) | a spawn request becomes an entity |
| [`xrServer_process_update.cpp`](xrServer_process_update.cpp.md) | a client's state report |
| [`xrServer_process_event.cpp`](xrServer_process_event.cpp.md) and its four `_event_*` siblings | the gameplay events a client may request |
| [`xrServer_perform_*.cpp`](xrServer_perform_transfer.cpp.md) | the authoritative acts: transfer, reject, destroy, migrate, respawn-point generation, game export |
| [`xrServer_perform_sls_*.cpp`](xrServer_perform_sls_load.cpp.md), [`xrServer_sls_clear.cpp`](xrServer_sls_clear.cpp.md) | level-state save, load, default and clear |
| [`xrServer_secure_messaging.cpp`](xrServer_secure_messaging.cpp.md) | the per-client shared secret |
| [`xrServer_updates_compressor.cpp`](xrServer_updates_compressor.cpp.md) | packing per-entity updates into broadcast packets |
| [`xrServer_info.cpp`](xrServer_info.cpp.md) | the server's description, logo and rules, sent to a joining client |
| [`xrServer_balance.cpp`](xrServer_balance.cpp.md) | team balancing |
| [`xrServer_svclient_validation.cpp`](xrServer_svclient_validation.cpp.md) | build-version agreement |
| [`xrServerMapSync.cpp`](xrServerMapSync.cpp.md) | telling a client where to download a level it lacks |

## State

```text
RECORD ClientRecord                       # extends the transport's own client record
  owner              : optional<ref server object>   # the entity this client controls
  net_ready          : bool               # has sent its first state report
  net_accepted       : bool               # admitted; broadcasts go only to accepted clients
  net_pass_updates   : bool               # forward this client's reports to the host client
  last_move_update   : int                # ms
  player_state       : optional<ref player state>    # owned; freed with the record
  ping_warnings      : (count : int, last_warned_at : int)
  admin              : (has_rights : bool, logged_in_at : int)
  key_digest         : text               # the copy-protection identity; see the pool
  secret_key         : bytes              # shared secret for authenticated messages
  last_key_seed      : int (32-bit)       # the seed of the outstanding key-sync request

RECORD Server
  entities        : map<int (16-bit), ref server object>   # every entity, by identifier
  respawn_queue   : sorted multiset of (due_at : int, entity : int (16-bit))
  cheaters        : list<(reason : text, client : ClientID)>
  delayed_packets : queue<(sender : ClientID, packet)>      # guarded by a lock
  id_allocator    : identifier allocator                    # see below
  seed_generator  : secure seed source
  game            : ref game-mode state                     # the rules; owns scoring
  disconnected    : ClientsPool                             # reconnect parking
  file_transfers  : optional<ref transfer site>
  updates         : update compressor
  server_info     : logo, rules, and a list of pending uploaders
```

**Invariants** — the entity table's key *is* the entity's own identifier; the two must agree
and a debug sweep checks it every message. Parent and child references between entities must
be mutually consistent: if an entity names a parent, that parent's child list contains it,
and vice versa. Those two invariants are the ones the debug verification exists to protect,
which makes them the ones a rebuild must hold.

**The entity identifier is 16 bits wide and that width is load-bearing**: it is the key here,
the field in every wire message, and the value in the save file. The maximum is two below the
type's maximum — one value is reserved as *invalid*, and one more is held out by the
allocator.

**The entity table's iteration order is load-bearing on load.** The source says so directly:
the player's own entity must be constructed first, and the original relies on the hash table
happening to produce that order on one platform. On the platform whose hash order differs,
an ordered map is substituted so the lowest identifier — the player's — comes first. **A
rebuild must make that explicit**: the load order is by entity identifier ascending, and
relying on a container's incidental order is the bug this works around rather than fixes.

### The identifier allocator

Parameterized rather than hard-coded, and the parameters are the decision:

```text
id_allocator: values 0 .. (max - 2), allocated in blocks of 256,
              one value reserved as "invalid",
              a freed identifier carries the time it was freed
```

**A freed identifier is not immediately reusable.** It carries a free timestamp, and the
allocator holds it back until enough time has passed. That is because a message naming a
destroyed entity may still be in flight; reusing the identifier at once would deliver it to
the wrong entity. The block size of 256 is an allocation-granularity choice with no further
consequence.

## Notes

**The network latency constant is 50 ms** and is declared here as a compile-time value. It
is the assumed one-way delay the server budgets for when scheduling anything that must appear
simultaneous to a client. A rebuild should measure it per client instead; the constant is a
2004 assumption.

The maximum client count bounds the screenshot-proxy array at twice that number — two
concurrent transfers per player — which is the only place the client cap leaks into an array
size.

The debug draw flags at the end of the header are a bitmask of which entity categories the
server-side debug overlay draws. They are development scaffolding and a rebuild may drop
them; they are listed because a reader of the original meets them and wonders.
