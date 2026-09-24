# src/xrGame/Level_network_map_sync.cpp

> The handshake that proves a joining client has the same level data as the server, and the loop that blocks level bring-up until the server has sent its game configuration.

**Needs** — [`Level.h`](Level.h.md) · [`Level_network_map_sync.h`](Level_network_map_sync.h.md) · [`xrServerMapSync.h`](xrServerMapSync.h.md) · [`MainMenu.h`](MainMenu.h.md) · [`xrNetServer/NET_Messages.h`](../xrNetServer/NET_Messages.h.md) · [`xrCore/stream_reader.h`](../xrCore/stream_reader.h.md) · [`xrEngine/XR_IOConsole.h`](../xrEngine/XR_IOConsole.h.md) · [Seam: Compression](../../SYSTEM-REQUIREMENTS.md#seam-compression)
**Used by** — [`Level_network_map_sync.h`](Level_network_map_sync.h.md)
**Tier floor** — T2: a streamed checksum over a hundred-megabyte file, then a polled state machine

## Purpose

Two clients running different versions of a map will disagree about geometry, and every
disagreement becomes a desync: bullets that hit on one machine and miss on the other,
players standing inside walls. The engine settles this before the match starts by having
the client announce the level it loaded — name, authored version string and a checksum of
the geometry file — and letting the server accept or reject it.

The second half of the file is the gate the level's load sequence sits in: level bring-up
cannot finish until the server has replied with the map verdict *and* with the game's
configuration, so `synchronize_map_data` is written as a function that is called
repeatedly and returns "done yet?".

## State

The record is declared in [`Level_network_map_sync.h`](Level_network_map_sync.h.md).

```text
RECORD LevelMapSyncData
  name                  : text     # the level currently loaded
  map_version           : text     # its authored version string
  level_geom_crc32      : int (32-bit)
  map_download_url      : text     # where the server says a missing map can be fetched
  sended_map_name_request : bool   # invariant: the request is sent at most once per attempt
  map_sync_received     : bool     # the server has answered
  map_loaded            : bool
  wait_map_time         : int      # poll iterations spent waiting; ~5 ms each
  invalid_geom_checksum : bool     # set by the message receiver, read by the gate
  invalid_map_or_version: bool     # same
```

**Invariants** — the two failure flags are written by the message-receiving path and read by
the polling path; they are the only channel between them. Sending the request clears both,
so a reconnect starts from a clean verdict.

## `CalculateLevelCrc32`

**Contract** — streams the level's geometry file in fixed-size blocks and folds each block's
checksum into a running value. Blocking; reads the whole file. Fails hard if the file
cannot be opened.

```text
FUNCTION CalculateLevelCrc32()
  buffer = 128 KiB of scratch
  reader = open "$level$/level.geom"
  crc = 0
  WHILE bytes remain
    read up to 128 KiB
    crc = crc XOR checksum(that block)      # note: XOR of block checksums, not a stream checksum
  END WHILE
  map_data.level_geom_crc32 = crc
```

**Notes** — the combination is an exclusive-or of independent per-block checksums, *not* a
checksum of the whole stream. That makes the result independent of block size, which is why
a client and a server with different buffer sizes still agree — but it also makes the
combination order-insensitive and far weaker than a real stream checksum: swapping two
blocks leaves it unchanged. It is a mismatch detector, not a tamper detector, and a rebuild
should treat it as such. Reproducing the exact value is only necessary to interoperate with
an original server.

The geometry file is the only file checksummed. It is the one whose contents must match for
collision and visibility to agree; textures and sounds may differ freely.

## `synchronize_map_data`

**Contract** — called repeatedly from the level's load sequence. Returns true when the
client may proceed, false to be called again. Sleeps briefly on each unsuccessful poll.
May trigger a reconnect and truncate the remaining load steps.

```text
FUNCTION synchronize_map_data() -> bool      # true = proceed
  IF NOT running as a client AND NOT recording a demo THEN
    # A listen server or single player is its own authority: nothing to verify.
    allow spawns; mark sync received
    RETURN synchronize_client()
  END IF

  map_data.CheckToSendMapSync()              # sends once
  pump the client receive queue

  IF waited more than ~5 seconds AND no answer AND not replaying a demo THEN
    reconnect; drop every remaining loading step but the first
    RETURN true                              # "done", because the load is being abandoned
  END IF

  IF no answer yet THEN sleep 5 ms; count the wait; RETURN false

  IF server said wrong map or version THEN
    reconnect; drop the remaining loading steps
    RETURN true
  END IF
  IF server said checksum mismatch THEN
    mark the connection down
    RETURN false                             # deliberately never completes: the load stalls
  END IF
  RETURN synchronize_client()
```

**Notes** — the two failure verdicts are handled differently and it is not obvious why. A
wrong map is recoverable by reconnecting (and the server has supplied a download address),
so the client retries. A checksum mismatch on the right map means the client's data is
altered; the connection is simply marked down and the function keeps returning false, which
leaves the loading screen up until the user backs out. That asymmetry is a decision, not an
oversight — but the stalled state has no user-facing message, which is a genuine defect a
rebuild should fix.

The timeout is counted in poll iterations, not in clock time. Each iteration sleeps about
five milliseconds, so a thousand iterations is roughly five seconds *if* the receive pump
is cheap. A rebuild should use the clock.

## `synchronize_client`

**Contract** — the second gate: asks the server once for the connection data, then returns
whether the game rules have been configured. Called repeatedly. On a listen server it also
pumps the server's own update, because one process is both ends.

```text
FUNCTION synchronize_client() -> bool
  IF the connection-data request has not been sent THEN send it reliably, mark sent
  IF game rules are configured THEN allow spawns; RETURN true
  IF this process also runs the server THEN
    pump the client receive queue
    step the server                  # otherwise the reply would never be produced
  END IF
  RETURN whether the rules are configured
```

**Invariants** — spawn messages are *denied* until this gate passes. That flag is the reason
the gate exists: a spawn that arrives before the game rules are known cannot be assigned to
a team, a respawn point or a score record.

**Notes** — the listen-server case stepping the server inside a client-side wait loop is the
clearest illustration of the single-process architecture: the client is blocking on a reply
that only it can cause to be produced. A rebuild that separates the two sides onto threads
or processes deletes this branch; a rebuild that keeps them in one loop must keep it.

## `LevelMapSyncData::CheckToSendMapSync`

**Contract** — sends the map announcement exactly once per connection attempt, reliably and
in order, and resets the verdict flags and the wait counter. Subsequent calls do nothing.

```text
FUNCTION CheckToSendMapSync()
  IF already sent THEN RETURN
  send { level name, map version, geometry checksum } as the map-name message
  sent = true; verdicts cleared; answer not received; wait counter zero
```

## `LevelMapSyncData::ReceiveServerMapSync`

**Contract** — decodes the server's one-byte verdict into the two failure flags and marks
the answer received. Called from the level's message dispatch.

```text
ENUM MapSyncResponse
  Ok
  InvalidChecksum        # right map, wrong bytes
  YouHaveOtherMap        # wrong map or wrong version
```

## `IsChecksumsEqual`

**Contract** — compares a supplied checksum against the locally computed one. The server
side of the same comparison; a pure predicate.
