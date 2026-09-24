# src/xrNetServer — client/server transport, message framing, bit-packed streams

Chapter 9 of the build order. Everything multiplayer rests on this module: the shape of a
message on the wire, the envelope that carries a batch of messages inside one datagram, the
client's path from "connect" to "in game", the server's roster of connected clients, and the
governor that decides how often either end is allowed to speak.

It rests on chapter 6 (`xrCore`) for the packet buffer, the configuration parser, the
compressor and the timer, and on nothing else. Nothing in the engine's single-player path
reaches it — but single-player still *runs through it*, because the engine always builds a
server and a client and connects them in-process (the `direct connect` mode, below).

---

## What this module is responsible for

Three things, and it is worth being precise about the boundaries between them because they
are separable and a rebuild may well separate them.

1. **The byte layout of a message.** A message is a flat byte buffer written field by field,
   with an explicit width and, for floats and directions, an explicit quantization. Both ends
   must produce and consume exactly the same sequence of widths. This layout lives in
   [`xrCore/net_utils.h`](../xrCore/net_utils.h.md) (the record type is used everywhere, so
   it sits in the core module) but it belongs to this chapter conceptually, and the rules are
   written out below.

2. **The envelope.** Many small messages are accumulated into one buffer, that buffer is
   compressed as a unit, and the result is handed to the transport as one datagram. Receipt
   reverses it. This is [`NET_Common.cpp`](NET_Common.cpp.md) plus
   [`NET_Compressor.cpp`](NET_Compressor.cpp.md).

3. **The session.** Who is connected, under what identity, how time is aligned between the
   two ends, how often each end may send, and what happens when a client stops answering.
   This is [`NET_Client.cpp`](NET_Client.cpp.md) and [`NET_Server.cpp`](NET_Server.cpp.md).

What this module is **not** responsible for: the meaning of any message. The message-type
identifiers, the per-entity update payloads, prediction and reconciliation all live in the
game module (chapter 23) and in the entity records (chapter 22). This module carries opaque
bytes and knows only the first 16 bits of each message — the type tag — and only so it can
log it.

---

## The seam this module sits on

The datagram transport itself is a [given seam](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport).
The original fills it with a Windows-only vendor library, which is the single reason
multiplayer does not port. Draw the line here, and draw it hard: **the vendor's concepts must
not appear in the contract.** What the engine actually demands of a transport is this and no
more.

```text
INTERFACE DatagramTransport

  # Lifecycle
  FUNCTION host(bind_port, session_name, max_clients, password : optional<text>,
                reserved_blob : bytes) -> result<Host, Error>
      # reserved_blob is attached to the session and readable by a prospective client
      # BEFORE it connects; the engine puts the map name, map version and a download
      # URL there so the client can refuse or fetch the level first.

  FUNCTION connect(host_address, bind_port, session_password : optional<text>,
                   identity_blob : bytes) -> result<Connection, Error>
      # identity_blob travels with the connection request and is handed to the server
      # when the connection is admitted; the engine puts the player name, the session
      # password and the client's process id there.

  FUNCTION discover(host_address, bind_port, probe : bytes, attempts, interval_ms,
                    timeout_ms) -> list<SessionDescription>
      # The engine can also skip discovery when it already knows the address.

  FUNCTION close(handle)

  # Traffic
  FUNCTION send(connection, payload : bytes, channel : ChannelFlags,
                timeout_ms) -> result<unit, Error>
  FUNCTION send_to(host, client, payload : bytes, channel : ChannelFlags, timeout_ms)
  FUNCTION pending_sends(connection_or_client) -> int   # depth of the outbound queue
  FUNCTION statistics(connection_or_client) -> LinkStatistics
  FUNCTION peer_address(client) -> (address : text, port : int)
  FUNCTION eject(client, reason : text)                 # server-initiated disconnect

  # Notifications, delivered on a transport-owned thread
  EVENT admit_request(peer_address) -> accept | reject_with(reason_text)
  EVENT client_created(client_handle, identity_blob)
  EVENT client_destroyed(client_handle)
  EVENT datagram(from, payload : bytes)
  EVENT session_terminated(reason_text)
  EVENT discovery_response(session_description)

RECORD ChannelFlags
  reliable       : bool   # retransmit until acknowledged
  ordered        : bool   # deliver in send order relative to other ordered sends
  high_priority  : bool   # jump the queue
  immediate      : bool   # do not let the engine coalesce this one; see below

RECORD LinkStatistics
  round_trip_ms        : int
  throughput_bps       : int
  peak_throughput_bps  : int
  packets_dropped      : int
  packets_retried      : int
  messages_sent        : int
  messages_received    : int
```

Everything on that list is satisfiable by any connection-oriented UDP library. Three details
are worth calling out because they are easy to miss when substituting one:

- **`immediate` is an engine concept, not a transport one.** The engine coalesces messages
  into an envelope itself; `immediate` means *flush my envelope now*, and the transport never
  sees the flag. A substitute transport should simply ignore it if it is passed through.
- **`pending_sends` is load-bearing.** The bandwidth governor (below) uses the outbound queue
  depth as its back-pressure signal. A transport that cannot report it forces the rebuild to
  invent another signal, and the pacing constants change meaning.
- **`reserved_blob` must be readable before the connection completes.** The client reads the
  map name out of it to decide whether it can join at all.

**The null filling.** [`empty/`](empty/README.md) is a second copy of the client and server
with every transport call deleted. It compiles everywhere, satisfies every link, and connects
to nothing: both `connect` and `host` walk their port-retry loop to the end and report
failure. It exists so the rest of the engine builds and links on platforms without the vendor
library. Treat it as the shape of the port, not as a working implementation — and see
[`empty/README.md`](empty/README.md) for what it does and does not preserve.

---

## Load-bearing idea 1 — the bit-packed stream

A message is a flat byte buffer with a cursor. Writing appends at the cursor; reading consumes
from a separate cursor. **There is no self-describing structure of any kind**: no field tags,
no lengths, no type markers. The reader knows what to read because it knows the message type,
which is the first field, and because reader and writer are the same codebase. This is what
makes the protocol *frozen against itself* and nothing else — a rebuild may redesign it
wholesale, but it may not mix old and new ends.

**The buffer.** Fixed capacity of 16384 bytes, which is also the largest datagram the receiver
will accept; a larger inbound one is discarded as hostile. Writes are bounds-checked against
that capacity, so an over-long message is a programming error, not a runtime condition.

**Byte order and alignment.** Little-endian, unaligned, no padding: a multi-byte field is the
target's native representation of that integer or float appended at whatever offset the cursor
happens to be. A 32-bit float is written as its exact bit pattern. A big-endian rebuild must
byte-swap at every field; nothing in the code does.

**The fixed-width primitives.** Signed and unsigned integers at 8, 16, 32 and 64 bits; a
32-bit float; three and four floats for a vector; twelve floats for a transform (three basis
rows and a translation, with the fourth column implied — `0,0,0,1`, never transmitted).

**There is no variable-length integer encoding.** Compactness comes from the *caller* choosing
the narrowest width that fits — entity identifiers are 16 bits, counts are 8, timestamps are
32 — and that per-field choice is part of the protocol just as firmly as the bit layout is.
A rebuild that "helpfully" widens a field to be safe has changed the wire format.

**Quantized real numbers.** Four forms, and their exact arithmetic matters because the
rounding error is visible in gameplay — it is what makes a remote player's aim jitter by a
fraction of a degree.

```text
# 16-bit quantized float over a caller-supplied range
FUNCTION write_q16(a : real, min : real, max : real)
  REQUIRE min <= a <= max                # violating this is a bug, not a clamp
  q = (a - min) / (max - min)            # normalize to [0,1]
  emit_u16( floor(q * 65535 + 0.5) )     # round to nearest

FUNCTION read_q16(min, max) -> real
  v = take_u16()
  RETURN v * (max - min) / 65535 + min   # exact inverse, up to float rounding

# 8-bit quantized float
FUNCTION write_q8(a, min, max)
  REQUIRE min <= a <= max
  q = (a - min) / (max - min)
  emit_u8( floor(q * 255 + 0.5) )

FUNCTION read_q8(min, max) -> real
  v = take_u8()
  RETURN (v / 255.0001) * (max - min) + min
  # The divisor is 255.0001 and not 255 on purpose: it keeps the largest code
  # strictly inside the range so the caller's range check cannot trip on the
  # round-trip. The cost is a systematic bias of one part in 2.55 million,
  # which is far below the 1/255 quantization step and therefore invisible.
```

Resolution follows directly: a 16-bit quantized value carries the range in 65535 equal steps,
an 8-bit one in 255.

```text
# Angles: normalized into [0, 2*pi) first, then quantized over [0, 2*pi]
FUNCTION write_angle16(a)  ->  write_q16(normalize_angle(a), 0, 2*pi)
FUNCTION write_angle8(a)   ->  write_q8 (normalize_angle(a), 0, 2*pi)
# Step: 2*pi/65535 ~= 9.59e-5 rad (0.0055 degrees) for the 16-bit form,
#       2*pi/255   ~= 0.0246  rad (1.41   degrees) for the 8-bit form.
# The 8-bit form is for things a player cannot aim with - a corpse's facing,
# a particle's spin. Anything a weapon points along uses the 16-bit form.
```

```text
# Unit directions: one 16-bit code, ~0.01 rad of angular error
FUNCTION write_direction(d : vector3)     -> emit_u16(compress_unit_vector(d))
FUNCTION write_scaled_direction(d)        # direction and length, separately
  mag = length(d)
  IF mag > tiny THEN unit = d / mag ELSE unit = (0,0,1); mag = 0
  write_direction(unit)
  write_float(mag)          # full 32-bit float: lengths are not quantized
```

The unit-vector code is the engine's own and is described where it lives
([`xrCore/_compressed_normal.h`](../xrCore/_compressed_normal.h.md)); its shape matters here
only as three facts a rebuild must reproduce: the top three bits are the three sign bits, the
remaining thirteen index a point on the octahedral triangle, and the all-zero code means the
zero vector. Decompression is a table lookup of 8192 normalization factors, built once at
startup.

**Strings.** Raw bytes followed by a terminating zero. No length prefix, no encoding
declaration — the bytes are whatever single-byte codepage the rest of the engine is using
(see [platform assumptions](../../SYSTEM-REQUIREMENTS.md#4-platform-assumptions)). A reader
that reads into a fixed buffer checks the length against it and treats an over-long string as
fatal; a reader that reads into a growable string does not. A null string is written as a lone
zero byte, indistinguishable from the empty string on read.

**Back-patched length prefixes.** Where a message carries a nested block whose length is not
known until it has been written, the writer reserves one byte (or two), writes the block, then
seeks back and fills in the length. The 8-bit form asserts the block is under 256 bytes and
the 16-bit form under 65536. This is the only backward seek in the format, and it is the only
reason writing needs random access to the buffer at all.

**The message header.** Every message begins with a 16-bit type identifier, written by
`begin` and consumed by `read_begin`. Reading also resets the read cursor to zero, so
`read_begin` is the canonical "start interpreting this message" step. The type identifiers
themselves belong to the game module.

**One incidental thing worth naming**, because it looks structural and is not: the packet
record can be pointed at a text-configuration writer instead of a byte buffer, so the same
field-by-field code that composes a message can instead dump it as readable key/value text.
That is a debugging affordance. It costs a branch per field in the original; a rebuild should
express it as a second implementation of one "field sink" interface rather than a flag inside
the buffer.

---

## Load-bearing idea 2 — the envelope

Messages are small — a position update is tens of bytes — and datagrams have a floor cost.
So the sender does not transmit messages; it transmits **envelopes**.

```text
ENVELOPE (what one datagram carries)
  tag            : u8     # 0xE1 = several messages inside, 0xE0 = exactly one
  unpacked_size  : u16    # total size of the payload BEFORE compression
  payload        : bytes  # compressed, see below

PAYLOAD when tag = 0xE1 (merged)
  repeated until unpacked_size bytes consumed:
    length : u16
    message: bytes[length]

PAYLOAD when tag = 0xE0 (single)
    message: bytes[unpacked_size]
```

The sender only ever produces the merged form. The single form is accepted on receipt and
never generated — a compatibility affordance with no live producer.

**Compression is per envelope and is opportunistic.** The payload is compressed with a
fast byte-oriented compressor, and if the result is not smaller than the input it is sent
uncompressed instead. A one-byte tag inside the payload says which happened, and a 32-bit
checksum of the compressed bytes follows it so a corrupt envelope is caught before it is
expanded into a buffer. Envelopes of 36 bytes or fewer skip the attempt entirely — below that
size the compressor's own header costs more than it saves. See
[`NET_Compressor.cpp`](NET_Compressor.cpp.md) for the full envelope-within-envelope layout.

**The accumulation rule** is the interesting part, because it is what keeps the two ends'
reliability guarantees honest:

```text
WHEN a message is queued for sending:
  IF the buffer would overflow the datagram limit
     OR the message's channel flags differ from those of the messages already buffered
     OR the message asks to go immediately
  THEN flush the buffer first
  append the message
  IF the message asks to go immediately THEN flush again
```

The middle condition is the load-bearing one. An envelope is transmitted with **one** set of
channel flags, so messages of different reliability or ordering may never share one. A rebuild
that batches indiscriminately would silently promote unreliable traffic to reliable (wasting
bandwidth) or demote reliable traffic (losing state changes).

There are **two** accumulation buffers per sender, not one, and a mode switch selects the
policy: by default everything shares one buffer; one mode strips the reliability flag off
everything before buffering; a third keeps reliable and unreliable traffic in separate buffers
so a burst of unreliable updates cannot delay a reliable state change behind it. The mode is a
runtime setting — evidence that no one policy was right for every deployment.

---

## Load-bearing idea 3 — the session

### Identity

A client is identified by a 32-bit opaque handle minted by the transport. The all-ones value
is reserved to mean *broadcast*. The engine never interprets the bits; it only compares them.
A rebuild is free to mint its own, subject to that one reserved value.

### The client's path from nothing to in-game

```text
STATE MACHINE (client)

  disconnected
    | connect(options)  - see the option string below
    v
  waiting                     # transport connected; engine has said nothing yet
    | server's sign-on packet arrives  (the two magic words, no payload)
    v
  completed                   # engine-level traffic is now accepted and queued
    | time synchronization runs on its own thread, concurrently
    v
  synchronized                # server-clock estimate is stable
    | ... game-layer handshake: the server describes the game mode, exports its
    |     state, and declares configuration finished; the client answers "ready"
    v
  in game
```

The first two transitions belong to this module; the last belongs to chapter 23. The hinge
between them is the sign-on packet, and it is deliberately trivial: two 32-bit constants and
nothing else. Until it arrives, inbound engine messages are **dropped**, not queued — that is
the rule that keeps a client from acting on state it has no context for.

Two other **system packets** share the same two-constant prefix and are recognized before any
engine message is: the sign-on, and the time-synchronization probe. They are distinguished
from engine messages purely by that prefix and from each other purely by their length. This is
fragile by construction — an engine message whose first eight bytes happened to match would be
swallowed — and a rebuild should give system traffic its own channel or its own tag byte
rather than reproducing the coincidence.

**The option string.** Connection parameters arrive as one slash-separated text blob, parsed
by substring search: the host name is everything before the first slash, then `psw=` (session
password), `name=` (player name), `pass=` (player password), `port=` (server port),
`portcl=` (client's own bind port). The server parses the same blob for `psw=`,
`maxplayers=`, `portsv=` and the marker `/single`. It is a command line in disguise, because
it *is* the console command's tail. A rebuild should pass a record; the only thing that has to
survive is the set of parameters.

**Port selection.** Both ends try their preferred port and, if it is busy, walk upward until
they succeed or run past the end of a 250-port window above a fixed base. If the port was
given explicitly, a busy port is fatal instead — an explicit port means someone is depending
on it. The window's width is an inheritance from the (now dead) matchmaking service, which
only scanned a bounded range.

### Time alignment

Client and server each have their own monotonic millisecond clock, and every timestamped
message means the *server's* clock. The client therefore maintains an offset.

```text
# Runs on its own thread from the moment of connection until it converges.
FUNCTION synchronize()
  clear sample buffer
  WHILE connected AND NOT synchronized
    WAIT until the outbound queue is empty        # else queueing delay pollutes the sample
    send probe { magic, magic, client_send_time }  # unreliable, unordered, high priority
    WAIT for a reply, up to 5 seconds
    IF sample count >= 256 THEN synchronized = true; adopt the current estimate

# The server's half: stamp and bounce. It does not compute anything.
ON probe received:  probe.server_time = now(); send it back on the same channel

# The client's half, per reply:
  round_trip = now() - probe.client_send_time
  offset     = probe.server_time + round_trip/2 - now()
  push offset into a ring buffer of 512 samples
  estimate   = mean(buffer)                       # rounded half-away-from-zero
  applied    = (applied * 5 + estimate) / 6       # first-order smoothing
```

Three decisions are worth carrying across. Sending only when the outbound queue has drained
is what makes the round-trip measurement mean *network* latency rather than *queue* latency.
Requiring 256 samples before declaring synchronization trades a slower entry into the game for
an offset that does not lurch. And the 5-in-6 smoothing means a single outlier moves the
applied offset by a sixth of its error and no more — the estimate is allowed to be noisy
because the thing derived from it is not.

The same offset update runs, without the thread, whenever the game layer receives a message
that carries the server's clock. On top of the computed offset sits a user-settable one,
added but never smoothed, so an operator can bias the whole thing by hand.

The offset is a *modular* quantity: the clocks are 32-bit millisecond counters and the
difference between them is taken as a signed 32-bit value that is allowed to wrap. A rebuild
using a wider clock must still subtract in the narrow type, or the estimate diverges after
about 25 days of uptime.

### The bandwidth governor

Neither end sends world updates whenever it likes. Both ask first, and the answer is the same
shape on both sides:

```text
FUNCTION has_bandwidth(peer) -> bool
  IF link is down THEN RETURN false
  interval = 1000 / configured_update_rate      # milliseconds; rate defaults to 30 per second
  IF "minimize updates" is set THEN interval = 1000
  IF now - peer.last_update_time <= interval THEN RETURN false   # too soon
  IF transport's pending-send count for this peer > allowed_backlog THEN
      peer.times_blocked = peer.times_blocked + 1
      RETURN false                              # the link is behind; do not pile on
  refresh peer's link statistics
  peer.last_update_time = now
  RETURN true
```

Two independent brakes: a **rate** brake (a fixed minimum interval between updates) and a
**depth** brake (never queue more when the transport is already holding more than a small
number of un-sent datagrams — 2 on the client, 3 on the server). The depth brake is the one
that matters on a bad link, and it is the reason `pending_sends` is a hard requirement of the
transport seam. A blocked attempt is counted, so a client that is being starved is visible in
the statistics rather than merely slow.

`minimize updates` collapses the rate to one per second. It exists for the case where a
client is present but not participating — spectating, or sitting in a menu.

In **direct-connect** mode (single-player and a listen server's own client) the governor keeps
the rate brake and drops the depth brake entirely, since there is no queue. Compression is
also skipped in that mode: both ends share a process and the bytes never leave it.

### The server's roster

The server holds one record per connected client, guarded by one lock, and hands out access
only through iteration and search callbacks — never a raw handle to the collection. The reason
is that the transport delivers events on its own thread while the simulation walks the roster
on the main one. A rebuild should keep the roster behind an interface for the same reason, in
whatever form its concurrency model prefers.

The roster's contract is stated in [`NET_PlayersMonitor.h`](NET_PlayersMonitor.h.md); the
non-obvious part is that a "send to every client" iteration takes a *second* lock, held across
the whole iteration, so that the composition of an outbound broadcast cannot interleave with
the handling of an inbound message.

### Admission, refusal and eviction

Admission is a three-stage funnel, and the stages happen at different layers:

1. **Before the connection is admitted at all**, the server is asked to vet the peer's
   address. It refuses banned addresses and, if a subnet allow-list is configured, addresses
   outside it. The refusal carries a human-readable reason back to the would-be client, which
   is the only message an un-admitted peer ever receives. The very first client to connect is
   exempt from the subnet check: that is the server's own local client, and on a listen server
   it arrives before the allow-list can mean anything.
2. **On admission**, the transport hands over the identity blob, the server mints its client
   record, and sends the sign-on packet.
3. **After sign-on**, the game layer runs its own checks — content authentication, build
   version, server password, key validation — and may evict.

**Bans** are (address, expiry) pairs persisted as a configuration file in the writable data
root, re-read at startup and rewritten on every change. Expiry is checked by sorting the list
by expiry and testing the last entry, which retires *at most one* ban per sweep; the sweep
runs often enough that this is not a bug, but it is a decision a rebuild should make
deliberately rather than inherit.

**The subnet allow-list** is a separate configuration file of CIDR blocks. It is stored sorted
and searched by binary search with a comparator that masks both operands — a small trick that
lets a plain address be looked up against a list of prefixes without a linear scan. An empty
list means "allow everyone", which is the deployed default. Note the addresses inside it are
held in network byte order while addresses elsewhere in the module are held in host order, and
a swap happens on the way in; a rebuild should pick one order and keep it.

**Eviction** is server-initiated and carries a reason string that reaches the client's
session-terminated hook. Disconnection in the other direction — a client that simply vanishes
— reaches the server as a transport event, at which point the record is marked disconnected,
the game layer is told, and the record is destroyed. There is **no timeout of the engine's
own**: a client that stalls without dropping its connection is the transport's problem to
notice, and the engine will happily wait. That is a real gap, and a rebuild whose transport
does not detect a dead peer must add one.

### Content authentication

A multiplayer server will not accept a client whose game data differs from its own. The check
is a single 64-bit number: the checksum of the merged configuration set, combined by
exclusive-or with the checksum of every file under a list of *important* paths, minus the
files under a list of *ignored* paths. The two lists are hard-coded in
[`NET_AuthCheck.cpp`](NET_AuthCheck.cpp.md) — configuration, scripts, shaders, weapon and
material sounds, the crosshair textures, and the engine's own modules are important;
localization, fonts, UI text, gameplay tables and the user's own data directory are ignored,
because those legitimately differ between installations.

Two honest observations about it. Combining per-file checksums with exclusive-or means the
result is order-independent, which is convenient, but it also means two files that swap
contents produce the same total, and a file whose checksum repeats cancels out. And the
ignore-list is matched by prefix while the important-list is matched by *substring*, so a path
containing `ui` anywhere is treated as important even outside the UI tree. Both are
reproducible facts about the original; neither is a good idea to reproduce.

---

## Reading order

Start with [`NET_Messages.h`](NET_Messages.h.md) — it is twenty lines and it defines the
channel vocabulary and the two system packets everything else refers to. Then
[`NET_Common.cpp`](NET_Common.cpp.md) for the envelope, then
[`NET_Server.cpp`](NET_Server.cpp.md) and [`NET_Client.cpp`](NET_Client.cpp.md) for the two
ends. The rest are support.

| File | Role |
|---|---|
| [`NET_Messages.h`](NET_Messages.h.md) | The channel-flag vocabulary and the two system packet layouts |
| [`NET_Shared.h`](NET_Shared.h.md) | Module-wide tunables, the debug-flag set, and the per-link statistics surface |
| [`NET_Common.h`](NET_Common.h.md) | Declares the envelope sender/receiver pair and the module's configuration switches |
| [`NET_Common.cpp`](NET_Common.cpp.md) | The envelope: accumulation, flags-change flush, merge and split |
| [`NET_Compressor.h`](NET_Compressor.h.md) | Declares the per-envelope compressor and its statistics |
| [`NET_Compressor.cpp`](NET_Compressor.cpp.md) | Opportunistic compression, the tag-and-checksum framing, size accounting |
| [`NET_Client.h`](NET_Client.h.md) | The client's surface: the state queries and the hooks a game layer overrides |
| [`NET_Client.cpp`](NET_Client.cpp.md) | Connection, sign-on, the time-synchronization loop, the client governor, the inbound queue |
| [`NET_Server.h`](NET_Server.h.md) | The server's surface, and the abstract operations a game layer must supply |
| [`NET_Server.cpp`](NET_Server.cpp.md) | Hosting, admission, broadcast, bans, the address record, the link statistics |
| [`NET_PlayersMonitor.h`](NET_PlayersMonitor.h.md) | The client roster and its locked-iteration contract |
| [`NET_AuthCheck.h`](NET_AuthCheck.h.md) | Declares the content-authentication path lists |
| [`NET_AuthCheck.cpp`](NET_AuthCheck.cpp.md) | Which files are checked and which are exempt, and why |
| [`ip_filter.h`](ip_filter.h.md) | Declares the subnet allow-list |
| [`ip_filter.cpp`](ip_filter.cpp.md) | Parsing CIDR blocks, and the masked-comparator binary search |
| [`NET_Log.h`](NET_Log.h.md) | Declares the packet trace log |
| [`NET_Log.cpp`](NET_Log.cpp.md) | The trace record, its buffering, and its stale message-name table |
| [`guids.cpp`](guids.cpp.md) | One-line compilation unit that materializes the vendor library's identifiers |
| [`stdafx.h`](stdafx.h.md) | Shared prelude; also where the session's application identifier lives |
| [`stdafx.cpp`](stdafx.cpp.md) | Prelude compilation unit |
| [`empty/`](empty/README.md) | The null transport filling for platforms without the vendor library |

---

## What is not recoverable from the source

Named here once so the twins need not repeat it:

- **The two magic constants** that mark a system packet (`0x12071980` and `0x26111975`) read
  as dates — 12 July 1980 and 26 November 1975 — almost certainly birthdays. They carry no
  structure; any two constants would do.
- **The 36-byte threshold** below which compression is not attempted has no derivation in the
  source. It is plausibly the size at which the compressor's own framing stops paying, but
  nothing measures it.
- **The 250-port search window** is a constraint of the dead matchmaking service, inherited
  rather than chosen.
- **The 256-sample synchronization requirement and the 512-sample ring** are stated without
  justification. The ring is twice the requirement, which suggests the numbers were chosen
  together, but the reason for either is gone.
- **Why the 8-bit quantizer reads with a divisor of 255.0001 while the 16-bit one reads with
  an exact 65535** is not explained; the effect is described above, the intent is inferred.
