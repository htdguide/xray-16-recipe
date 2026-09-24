# src/xrGame/Level_network_Demo.cpp

> Records a multiplayer session by logging every server message to a file with its arrival time, and replays it by feeding those messages back into the client at the same relative times, so that a recording is indistinguishable from a connection.

**Needs** — [`Level_network_Demo.h`](Level_network_Demo.h.md) · [`Level.h`](Level.h.md) · [`Message_Filter.h`](Message_Filter.h.md) · [`DemoPlay_Control.h`](DemoPlay_Control.h.md) · [`DemoInfo.h`](DemoInfo.h.md) · [`Spectator.h`](Spectator.h.md) · [`xrServer.h`](xrServer.h.md) · [`game_sv_mp.h`](game_sv_mp.h.md) · [`game_cl_mp.h`](game_cl_mp.h.md) · [`UIGameDM.h`](UIGameDM.h.md) · [`xrEngine/CameraManager.h`](../xrEngine/CameraManager.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — [`Level_network_Demo.h`](Level_network_Demo.h.md)
**Tier floor** — T1: the header and per-packet record are written as raw memory images with no padding, and playback seeks by byte offset in a stream

## Purpose

The demo system is built on one idea: a multiplayer client is a pure function of the
stream of server messages it receives. Record that stream with timestamps and you can
replay the entire match by injecting the same bytes into the same receive entry point,
with no server involved and no special cases inside the game code. Everything in this file
serves that idea — capture on the way in, a file format that preserves arrival order and
timing, and a playback clock that releases packets when their recorded time has come.

The consequences are the interesting part. Because playback drives the client and not the
server, the viewer is not a player: a fake spectator entity has to be conjured so there is
something to attach a camera to. And because restarting playback means rewinding the byte
stream, the position of the first spawn message must be remembered on the first pass.

## State

```text
RECORD DemoState                     # lives on the level
  saving         : bool              # recording
  save_started   : bool              # header written, packets are being appended
  playing        : bool              # a demo file is open for playback
  play_started   : bool              # playback clock is running
  play_stopped   : bool              # the stream reached its end
  start_global   : int               # the frame clock value playback started at
  spectator      : object            # the fake entity the camera follows
  msg_filter     : MessageFilter     # optional tap on the replayed message stream
  play_control   : DemoPlayControl   # the viewer's transport controls
  writer / reader                    # exactly one is non-empty at a time
  header         : DemoHeader
  info_file_pos  : int               # where the fixed-size info block sits in the file
  info           : DemoInfo          # the match summary, or empty if not yet read/written
  prev_packet_pos    : int           # byte offset of the packet most recently examined
  prev_packet_dtime  : int           # its recorded time offset
  starting_spawns_pos   : int        # byte offset of the first spawn message
  starting_spawns_dtime : int        # its time offset
```

**Invariants**

- `saving` and `playing` are mutually exclusive, asserted at both entry points. The
  predicates the rest of the engine asks — "is a demo playing", "is a demo being saved" —
  are each defined as *one true and the other false*, so a corrupted state answers no to
  both rather than yes to both.
- The info block is written at a **fixed maximum size** at a **known offset**, and the
  writer seeks past it before the first packet. The summary of a match is only knowable
  when the match ends, but it must be readable from the front of the file without scanning
  it, so space is reserved up front and filled in later. This is the reason the format has
  a maximum info size at all.
- `starting_spawns_pos` is zero until the first spawn message has been seen. Restart
  refuses while it is zero, because there is nothing to rewind to.

## File format

```text
RECORD DemoHeader                    # written as a raw image, no padding
  time_global       : int (32-bit)   # the frame clock when recording began
  time_server       : int (32-bit)   # the server clock at the same instant
  time_delta        : int (32-bit, signed)
  time_delta_user   : int (32-bit, signed)

# then: the server's options string, length-prefixed
# then: the match info block, padded to a fixed maximum size
# then: a sequence of

RECORD DemoPacket                    # written as a raw image, no padding
  time_global_delta : int (32-bit)   # frame-clock offset from the header's time_global
  time_receive      : int (32-bit)   # the receive stamp the transport gave the packet
  packet_size       : int (32-bit)
  # followed by packet_size bytes: the message exactly as it arrived
```

The two clock deltas in the header exist because the client keeps an estimate of its
offset from the server's clock and a user-tunable correction on top of it. Replaying
without restoring both makes every server timestamp inside the recorded messages land at
the wrong time.

`time_receive` is recorded and restored into the replayed packet but nothing reads it;
this is a field carried for symmetry with the live path.

## `PrepareToSaveDemo`

**Contract** — opens a demo file for writing and enters the saving state. Names the file
from the local wall clock, in the log directory. Fails hard if a demo is already playing.
Allocates a writer that is only released by `StopSaveDemo`.

**Notes** — the timestamped filename is the only naming scheme; there is no way to choose
a name. The two platform spellings of "format the local time" produce different filenames
(one is a numeric stamp, the other the platform's human-readable time string, which
contains spaces and a trailing newline) — a rebuild should pick one canonical form.

## `StartSaveDemo`

**Contract** — writes the header and the server options, reserves the info block's space,
and begins appending packets. Separate from `PrepareToSaveDemo` because the server's
options are only known once the connection has been negotiated, which is after the file
must already exist.

## `SaveDemoHeader`

**Contract** — snapshots the four clock values and the server options to the front of the
file, records where the info block will go, and seeks past its reserved size.

## `SaveDemoInfo`

**Contract** — fills in the reserved info block from the live game state, then returns the
write position to where it was. Only meaningful in a multiplayer game; returns silently
otherwise. Idempotent-ish: called repeatedly during a match, each call overwrites the
block with a fresher summary.

**Invariants** — the write position on exit equals the write position on entry, or the
packet stream is corrupted from that point on.

## `SavePacket`

**Contract** — appends one received server message: its frame-clock offset from the header,
its receive stamp, its length and its bytes. Called from the client's message receive path
whenever recording is active. Does not block on anything but the file write.

```text
FUNCTION SavePacket(packet)
  write (frame_clock_now - header.time_global)
  write packet.time_receive
  write packet.length
  write packet.bytes
```

## `PrepareToPlayDemo`

**Contract** — opens a named demo file from the log directory for streamed reading, parses
its header, and enters the playing state. Returns false and logs on a missing file or a
header that does not leave at least one packet's worth of stream behind it. Fails hard if
recording is in progress.

## `LoadDemoHeader`

**Contract** — reads the clock header, the server options string and the info block, then
seeks to the end of the info block's reserved space. Returns whether at least one packet
header remains.

**Invariants** — must be called exactly once per opened file; asserts that no info block
has already been loaded.

## `StartPlayDemo`

**Contract** — starts the playback clock. Clears the spectator, stamps the start of the
frame clock, resets playback speed to real time, clears the remembered spawn position, and
installs the one-shot filter that will capture it.

```text
FUNCTION StartPlayDemo()
  REQUIRE playing AND NOT play_started
  spectator       = none
  play_started    = true
  start_global    = frame_clock_now
  set play speed  = 1
  starting_spawns_pos   = 0
  starting_spawns_dtime = 0
  CatchStartingSpawns()
```

## `SimulateServerUpdate`

**Contract** — the per-frame heart of playback. Releases every packet whose recorded time
offset has been reached by the elapsed playback time, in order, into the client's ordinary
message handler. Called once per frame while a demo is playing.

```text
FUNCTION SimulateServerUpdate()
  elapsed = frame_clock_now - start_global
  WHILE LoadPacket(packet, elapsed)
    IF msg_filter exists THEN msg_filter.observe(packet)   # taps, does not consume
    deliver packet to the client's normal message entry point
  END WHILE
```

**Notes** — the delivery target is the *same* entry point the network transport calls.
That is the whole design: no code below this line knows whether it is connected to a server
or reading a file. A rebuild must preserve that seam or it will end up with a second,
divergent copy of the client's message handling.

Playback speed is implemented by scaling the device's time factor, so `elapsed` advances
faster or slower than wall time and packets are released accordingly — which also slows
down animation, physics and sound, which is what a viewer expects from slow motion.

## `LoadPacket`

**Contract** — reads the next packet header; if its time offset has arrived, reads the body
into the destination and returns true; otherwise rewinds to the packet header and returns
false. Records the packet's file position and time offset in either case, which is what
makes the spawn-capture filter able to remember where a message came from. Stops playback
when the packet it just consumed was the last one. Asserts a sane packet size — a demo file
is untrusted input and an absurd length would otherwise overrun the destination buffer.

```text
FUNCTION LoadPacket(dest, elapsed) -> bool
  IF no reader OR at end of stream THEN RETURN false
  prev_packet_pos = current position
  read header
  prev_packet_dtime = header.time_global_delta

  # Strictly-less normally; less-or-equal while a map-name request is outstanding, so
  # that the packets stamped at exactly the current instant are not held back during
  # the map handshake.
  due = awaiting map name ? (header.time_global_delta <= elapsed)
                          : (header.time_global_delta <  elapsed)
  IF NOT due THEN
    seek back to prev_packet_pos
    RETURN false
  END IF
  REQUIRE header.packet_size < MAX_PACKET_SIZE
  read header.packet_size bytes into dest; reset dest read cursor
  IF fewer than one packet header remains THEN StopPlayDemo()
  RETURN true
```

**Notes** — the boundary condition (`<` versus `<=`) is a genuine seam and is worth keeping:
at the very start of playback the elapsed time is zero, so a strict comparison would never
release the packets recorded at offset zero and the map handshake would deadlock.

## `CatchStartingSpawns` and `MSpawnsCatchCallback`

**Contract** — together, a one-shot observer that records where in the file the first entity
spawn message lives, and at what time offset, then removes itself.

```text
FUNCTION CatchStartingSpawns()
  install a filter on SPAWN messages calling MSpawnsCatchCallback

FUNCTION MSpawnsCatchCallback(message, subtype, packet)
  starting_spawns_pos   = prev_packet_pos      # the packet currently being delivered
  starting_spawns_dtime = prev_packet_dtime
  remove the SPAWN filter                       # one shot
```

**Notes** — this is why `LoadPacket` bothers to remember the previous packet's position: the
filter sees a decoded message, not a file offset, and needs the reader's bookkeeping to
translate one into the other. The pair is the only reason the message filter exists on the
playback path at all in a shipping build.

## `RestartPlayDemo`

**Contract** — rewinds playback to the first spawn message, destroying and re-creating the
entire world in the process. Refuses if no demo is playing or the spawn position was never
captured.

```text
FUNCTION RestartPlayDemo()
  IF NOT playing OR starting_spawns_pos == 0 THEN log error; RETURN
  IF play_started THEN
    remove_objects()                 # MUST run while still in the started state
    StopPlayDemo()
  END IF
  play_started = true; play_stopped = false
  # Rewind the clock by exactly the amount the spawn packet was stamped with, so the
  # replayed stream lands on the same schedule it did the first time.
  start_global = frame_clock_now - starting_spawns_dtime
  seek reader to starting_spawns_pos
  set play speed = 1
```

**Invariants** — the world teardown must happen *before* the state flips out of "started".
The teardown runs a number of client update passes as objects unwind, and those passes
consult the demo state; tearing down after the flip makes them take the not-playing branch.

**Notes** — the debug counters reset alongside the teardown exist because the camera and the
spectator both assert they are updated at most once per frame, and the extra unwinding
passes trip that assertion. That is a symptom of the ordering above, not a separate rule.

## `StopPlayDemo` · `StopSaveDemo`

**Contract** — end playback (clock to real time, started false, stopped true; the reader is
deliberately *not* released so that a restart is still possible) and end recording (close
the writer).

## `SpawnDemoSpectator`

**Contract** — creates the fake entity a demo viewer inhabits: a spectator spawned through
the server side of the multiplayer game rules, named after the local player, assigned a
respawn point, spawned, and then its server-side record destroyed. Requires an active
server game state and a local player record.

**Invariants** — the spawn flags mark it local, player-class and *phantom*; the phantom flag
exists purely to let the rest of the code recognize this spectator as fake and not count it
as a participant.

**Notes** — the server-side record is destroyed immediately after the spawn is issued. This
is the general shape of a transient spawn in this engine: the server object is a message
generator, and once its spawn message has been produced the live client object is the only
thing that needs to exist. A rebuild whose server objects are the authority for the whole
session must instead keep it and mark it non-persistent.

## `SetDemoSpectator` · `GetDemoSpectator`

**Contract** — set and read the entity the demo camera follows; setting asserts the object
really is a spectator. This is the control entity in all but name, kept separately because
the demo viewer is not a connected player.

## `GetDemoPlayPos`

**Contract** — playback progress as a fraction, computed from the byte offset in the file
rather than from time. Returns 1 at end of stream.

**Notes** — byte position is not time position: a quiet stretch of the match occupies few
bytes and a firefight many, so the progress bar advances unevenly. The commented-out seek
implementation is why — seeking by fraction requires walking packet headers to accumulate
the time shift, and the accumulated result disagrees with the fraction it was asked for.

## `GetDemoPlaySpeed` · `SetDemoPlaySpeed`

**Contract** — read and write the playback rate as the device's time factor. Refuses if
playback has not started, and caps the rate at eight times real time. Above that, packets
arrive faster than the client can apply them and playback diverges from what was recorded.

## `GetMessageFilter` · `GetDemoPlayControl`

**Contract** — lazily create and return the message tap and the viewer's transport control
object. Never return empty. Created on first use because a session that neither records nor
plays needs neither.
