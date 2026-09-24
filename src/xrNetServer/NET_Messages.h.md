# src/xrNetServer/NET_Messages.h

> The channel vocabulary — how a caller asks for reliability, ordering and priority — and the
> layout of the two packets the transport layer sends on its own behalf.

**Needs** — [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — [`Level_network_digest_computer.cpp`](../xrGame/Level_network_digest_computer.cpp.md) · [`Level_network_map_sync.cpp`](../xrGame/Level_network_map_sync.cpp.md) · [`Level_network_spawn.cpp`](../xrGame/Level_network_spawn.cpp.md) · [`Level_network_start_client.cpp`](../xrGame/Level_network_start_client.cpp.md) · [`Level_secure_messaging.cpp`](../xrGame/Level_secure_messaging.cpp.md) · [`RocketLauncher.cpp`](../xrGame/RocketLauncher.cpp.md) · [`alife_simulator_script.cpp`](../xrGame/alife_simulator_script.cpp.md) · [`alife_switch_manager.cpp`](../xrGame/alife_switch_manager.cpp.md) · [`alife_update_manager.cpp`](../xrGame/alife_update_manager.cpp.md) · [`console_commands_mp.cpp`](../xrGame/console_commands_mp.cpp.md) · [`file_transfer.cpp`](../xrGame/file_transfer.cpp.md) · [`filetransfer_common.h`](../xrGame/filetransfer_common.h.md) · [`game_cl_base.cpp`](../xrGame/game_cl_base.cpp.md) · [`game_cl_capture_the_artefact.cpp`](../xrGame/game_cl_capture_the_artefact.cpp.md) · _and 20 more_
**Tier floor** — T1: it describes two structures that are transmitted as raw memory images,
field for field, with no padding.

## Purpose

Two unrelated things share this file because both are the transport layer speaking for itself
rather than for the game. The first is the translation from *what the caller wants* (reliable?
in order? urgent?) to whatever the transport calls those things. The second is the pair of
packets that the client and server exchange about the connection itself, which no game code
ever sees.

Keeping the channel translation in one place is the point: it is the only file that names the
transport's own flag constants, so replacing the transport touches this file and nothing else.

## State

Stateless — declarations only.

## `net_flags`

**Contract** — takes four independent wishes and returns the opaque channel descriptor the
transport understands. The defaults are the interesting part, because most call sites take
them: **unreliable, ordered, normal priority, batched**. A caller that wants delivery
guaranteed must say so.

```text
RECORD ChannelFlags
  reliable      : bool   # default false - retransmit until acknowledged
  ordered       : bool   # default true  - keep send order among ordered sends
  high_priority : bool   # default false - jump the transport's queue
  immediate     : bool   # default false - flush the envelope now, do not batch

FUNCTION channel(reliable, ordered, high_priority, immediate) -> ChannelFlags
```

**Notes** — the `immediate` wish is *not* a transport concept. It is a bit the engine sets in
the same word for convenience and strips out again before handing the word to the transport;
its only reader is the envelope accumulator in [`NET_Common.cpp`](NET_Common.cpp.md), which
treats it as "flush". A rebuild should carry it as a separate argument rather than smuggling
it through the transport's flag word.

On a platform with no transport the function returns a zero descriptor, which is why the null
filling in [`empty/`](empty/README.md) still compiles: every call site keeps working and every
channel becomes the same do-nothing channel.

## `MSYS_CONFIG`

**Contract** — the server's sign-on packet. It is sent once per client, reliably and
immediately, the moment the server has accepted the connection and built its client record.
Until the client receives it, the client drops every engine message that arrives. It carries
no payload: its arrival *is* the payload.

```text
RECORD SignOnPacket          # exactly 8 bytes, no padding
  magic_a : int (32-bit)     # 0x12071980
  magic_b : int (32-bit)     # 0x26111975
```

## `MSYS_PING`

**Contract** — the time-synchronization probe, sent by the client and bounced by the server.
The client fills its own send time; the server overwrites the middle field with its own clock
and returns the packet unchanged on an unreliable, unordered, high-priority, immediate
channel. The third timestamp field is never written by anyone.

```text
RECORD ProbePacket           # exactly 20 bytes, no padding
  magic_a          : int (32-bit)   # 0x12071980 - same two constants
  magic_b          : int (32-bit)   # 0x26111975
  client_send_time : int (32-bit)   # client's monotonic clock, milliseconds
  server_time      : int (32-bit)   # filled by the server on the bounce
  client_recv_time : int (32-bit)   # declared, never written, never read
```

**Notes** — the receiver tells a system packet from an engine message by testing the first two
words against the magic pair, then tells the two system packets apart **by length alone**: 8
bytes means sign-on, 20 bytes means a probe. Nothing else distinguishes them. This is a real
fragility — an engine message that happened to begin with those eight bytes would be
misrouted — and a rebuild should give system traffic its own tag byte or its own channel.

The unused third timestamp is a leftover from a symmetric round-trip scheme that was
abandoned: the offset is computed from the client's own receive clock at the moment of
delivery, not from a field. It costs four bytes per probe and nothing else.

Both records are transmitted as raw memory images, so their field order, widths and the
absence of padding are all load-bearing — but only against *themselves*, since both ends are
this codebase. A rebuild may redesign them freely.
