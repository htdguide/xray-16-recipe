# src/xrNetServer/NET_Client.cpp

> The client end of a session: establishing the connection, waiting for sign-on, aligning the
> client's clock to the server's, pacing outbound updates, and queueing inbound messages for
> the simulation to drain.

**Needs** — [`NET_Client.h`](NET_Client.h.md) · [`NET_Common.h`](NET_Common.h.md) · [`NET_Messages.h`](NET_Messages.h.md) · [`NET_Log.h`](NET_Log.h.md) · [`NET_Server.h`](NET_Server.h.md) · [`xrCore/net_utils.h`](../xrCore/net_utils.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport) · [Seam: Threads](../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — reached through its declarations in [`NET_Client.h`](NET_Client.h.md); callers name that, not this file.
**Tier floor** — T1: it reinterprets received datagrams as system-packet memory images and
runs a clock estimator over wrapping 32-bit arithmetic.

## Purpose

Everything a client must do that is not about the *meaning* of messages. Three jobs that are
genuinely separate and only share a file because they share a connection: get connected, know
what time it is on the server, and decide when it is allowed to speak. A rebuild may split
them; the connection state and the clock offset are the only state they share.

## State

```text
ENUM ConnectionState
  fails        # no connection, or the attempt failed
  waiting      # transport is connected; the server has not signed on yet
  completed    # sign-on received; engine messages are accepted

RECORD ClientSession
  state              : ConnectionState
  synchronized       : bool      # the clock estimate has converged
  disconnected       : bool      # the session was terminated from the far end
  client_id          : int (32-bit)   # assigned by the transport
  inbound            : MessageQueue
  statistics         : LinkStatistics
  last_update_time   : int (32-bit, wraps)   # governor bookkeeping
  time_offset        : int (32-bit, wraps)   # smoothed server-minus-client offset
  offset_estimate    : int (32-bit, wraps)   # latest unsmoothed mean
  offset_user        : int (32-bit, wraps)   # operator bias, added, never smoothed
  session_description: GameDescription       # map name, version, download URL
  discovered_hosts   : list<SessionDescription>

RECORD OffsetSamples               # a ring, shared by the sync thread and the receive path
  table : list<int (32-bit)>       # capacity 512
  write : int                      # next slot
  count : int                      # saturates at capacity

CONSTANT samples_required = 256    # before declaring the clock synchronized
CONSTANT sample_wait_ms   = 5000   # how long one probe waits for its reply
```

**Invariants** — while the state is not `completed`, every inbound engine message is
discarded. System packets are recognized and acted on in every state. The two are told apart
by the magic prefix described in [`NET_Messages.h`](NET_Messages.h.md).

`time_offset` is a signed difference between two wrapping 32-bit millisecond clocks, and it is
required to wrap. A rebuild that widens the clock must still take the difference in the narrow
type.

## `INetQueue`

**Contract** — a hand-off queue of received messages between the transport's delivery thread
and the simulation. Producers take a message buffer, fill it and it becomes visible in order;
the consumer peeks the oldest, reads it, and releases it. Message buffers are recycled rather
than freed: the queue keeps a free list of them, seeded with sixteen, and grows on demand.
Locking is explicit and the caller brackets its whole drain, so the simulation sees a
consistent batch rather than a moving target.

```text
FUNCTION acquire() -> MessageBuffer         # take from the free list, or make one
FUNCTION peek()    -> optional<MessageBuffer>   # the oldest ready message
FUNCTION release()                          # retire the oldest, returning it to the free list
FUNCTION lock() / unlock()                  # the caller brackets its whole drain
```

**Notes** — message buffers are 16 kilobytes each, so the free list is real memory and the
code shrinks it: whenever the queue is touched and the free list holds more than 32 buffers
and none has been created in the last 60 seconds, one buffer is destroyed. That is a
decay rule, and the decision it encodes — *a burst should not pin its peak footprint forever*
— is worth keeping even though the mechanism (a global "time of last creation" that every
queue shares) is not.

The lock is taken by the producer path and by the explicit bracket, but *not* by `peek` and
`release`, which rely on the caller holding it. In the original several of those internal
lock acquisitions are commented out — the locking was moved outward and the inner calls were
left behind as comments. A rebuild should state the ownership once: the consumer holds the
lock across its entire drain.

Copying a message into the queue copies the whole fixed-size buffer, including the unused
tail. That is 16 kilobytes per message regardless of the message's length — measurably
wasteful, and only survivable because the copying variant is rarely used.

## `Connect`

**Contract** — parses the option string, creates the transport endpoint, finds the server,
and completes the transport-level connection. Returns whether it succeeded. Blocking, and
takes as long as the port search and discovery need. On success the session is in `waiting`,
not `completed`: the server must still sign on. Failure paths call one of four refusal hooks
so the user interface can say what went wrong.

```text
FUNCTION connect(options : text) -> bool
  IF direct_connect THEN RETURN true          # no transport at all; see below

  server_host = options up to the first '/'
  session_password = value of "psw="          # the server's password
  player_name      = value of "name="
  player_password  = value of "pass="
  server_port      = value of "port="   , default the LAN base + 1
  client_port      = value of "portcl=" , default the LAN base + 2, remembered as explicit

  state = waiting; synchronized = false; disconnected = false

  create transport endpoint
  identity = { player_name, player_password, this process's identifier }
  attach identity to the endpoint      # the server reads it on admission

  IF server_host is "localhost" THEN
    # The server is in this process. No discovery: connect straight at it,
    # walking the client port upward until one binds.
    FOR port FROM client_port WHILE connect fails
      IF the port was given explicitly THEN FAIL WITH port_busy
      IF port passes the end of the LAN window THEN FAIL WITH no_free_port
    read the session's attached description; IF absent THEN invalid_host hook; FAIL
  ELSE
    # Discover first: 10 probes, one second apart, one second of patience each.
    FOR port FROM client_port WHILE discovery fails AND port within range
      probe the server address with the marker "ToConnect"
      ON invalid address     -> invalid_host hook;  FAIL
      ON session full        -> session_full hook;  FAIL
    IF no host answered THEN invalid_host hook; FAIL
    connect to the first host that answered
    ON wrong password -> invalid_password hook; FAIL
    ON session full   -> session_full hook;     FAIL

  time_offset = 0
  RETURN true
```

**Notes** — the option string is a console command's tail, parsed by substring search. Every
value ends at the next slash or at the end of the string, so no value may contain a slash.
That is a constraint a rebuild inherits only if it inherits the format, and it should not.

The port walk exists because two clients on one machine cannot share a bind port, and because
a fixed port would collide with anything else. An *explicit* port that is busy is fatal rather
than skipped, which is the right distinction: an explicit port means someone downstream is
depending on it. The upper bound of the walk is 250 ports above a fixed base — a limit
inherited from the matchmaking service, which only ever scanned that range.

The local case is not merely a shortcut. It skips discovery entirely and reads the session
description directly off the connected endpoint, because a listen server's own client connects
before the server is advertising anything.

Discovery returns a *list* of sessions and the client connects to the first. The list, and
the code that de-duplicates responses by session identifier, are the remains of a server
browser that now lives elsewhere.

In direct-connect mode this function does nothing at all: there is no transport, the server
delivers the sign-on packet by calling the client's receive path directly, and the rest of the
state machine runs unchanged. That is the single most useful property of this design — *the
client does not know whether there is a network* — and a rebuild should preserve it, because
it is what lets single-player and multiplayer share one code path.

## `_Recieve` — the inbound classifier

**Contract** — the first thing every inbound datagram meets, after the envelope has been
split. Decides whether the bytes are a system packet or an engine message, and in the latter
case whether the session is far enough along to care. Runs on the transport's thread.

```text
FUNCTION on_message(data, size)
  statistics.bytes_in = statistics.bytes_in + size

  IF size >= 8 AND data begins with the two magic words THEN
    IF size = probe_packet_size THEN               # 20 bytes
      now        = client clock
      round_trip = now - data.client_send_time
      offset     = data.server_time + round_trip/2 - now
      push offset into the sample ring
      recompute the estimate
      RETURN
    IF size = sign_on_packet_size THEN             # 8 bytes
      state = completed
      RETURN
    report an unknown system message and RETURN

  IF state = completed THEN
    optionally trace the message
    hand it to the queue
  # otherwise: dropped, silently
```

**Invariants** — the offset is computed against the client's clock read *at delivery*, so the
measurement includes the time the datagram spent in the transport's receive path. That is
accepted; what the sync thread controls for is the *send* side, by refusing to probe while the
outbound queue is non-empty.

**Notes** — engine messages arriving before sign-on are dropped rather than queued. This is
the rule that stops a client from acting on world state before it has been told which world it
is in.

## `OnMessage` — enqueue

**Contract** — copies a received message into a queue buffer, stamps it with the current
server-time estimate, and reads its type tag so the tag is available without re-parsing. The
timestamp is the *server's* time, not the client's: everything downstream reasons in server
time.

## `SendTo_LL` / `Send` / `Flush_Send_Buffer`

**Contract** — `Send` queues a message into the envelope accumulator;
`Flush_Send_Buffer` closes the current envelope; `SendTo_LL` is the accumulator's transmit
operation and is the only place in the client that touches the transport. A send on a
disconnected session is a no-op, not an error. Failures are logged and swallowed — by the
time a datagram fails to leave, the session is already over and the termination event is on
its way.

## `net_HasBandwidth` — the client governor

**Contract** — asked before composing a world update. Returns whether the client may send.
Consulting it has a side effect: a `true` answer records the send time, so the caller must
actually send.

```text
FUNCTION has_bandwidth() -> bool
  IF disconnected THEN RETURN false
  interval = 1000 / update_rate                 # default rate 30 per second
  IF minimize_updates THEN interval = 1000
  IF update_rate = 0 THEN RETURN false          # rate zero means "never"
  IF now - last_update_time <= interval THEN RETURN false

  IF direct_connect THEN                        # no queue to watch
    last_update_time = now
    RETURN true

  pending = transport's outbound queue depth
  IF pending > pending_limit THEN               # default 2
    statistics.times_blocked = statistics.times_blocked + 1
    RETURN false
  refresh statistics
  last_update_time = now
  RETURN true
```

**Notes** — that the query mutates `last_update_time` makes it a *reservation*, not a
question, and a caller that asks and then decides not to send has silently skipped a slot. A
rebuild should either make that explicit or split the reservation out.

The comment in the original says the reduced rate is "approximately three times per second"
where the code makes it one; the comment is wrong and the code is what ships.

## `Sync_Thread` / `Sync_Average` / `net_Syncronize`

**Contract** — `net_Syncronize` clears the samples and starts a dedicated thread; the thread
probes until the estimate converges and then exits. `Sync_Average` recomputes the estimate
from the ring and is called from the receive path as each reply lands, so the estimate is
current between probes.

```text
FUNCTION sync_loop()                 # its own thread, exits when synchronized or disconnected
  clear samples
  WHILE connected AND NOT disconnected
    IF synchronized THEN BREAK
    WAIT until the transport's outbound queue is empty    # else queue delay pollutes the sample
    send a probe { magic, magic, now } unreliable, unordered, high priority
    remember the sample count; WAIT up to 5 seconds for it to grow
    IF sample count >= 256 THEN
      synchronized = true
      adopt the unsmoothed estimate as the offset      # jump once, then never again

FUNCTION recompute_estimate()
  estimate = mean of the ring, rounded half away from zero
  applied  = (applied * 5 + estimate) / 6              # first-order smoothing
```

**Invariants** — the ring holds 512 samples and convergence needs 256, so at the moment of
convergence the ring is half full and the mean is over every sample taken. Afterwards the ring
keeps filling and eventually overwrites, and the estimate becomes a moving average over the
last 512 replies.

**Notes** — the one-time jump at convergence is the interesting decision. Before it, the
smoothed offset is still climbing towards the truth and using it would make the client's idea
of server time creep; adopting the unsmoothed mean in one step means the client enters the
game with a correct offset and only ever drifts by sixths afterwards.

Waiting for the outbound queue to drain before each probe is what makes the round-trip
measurement mean network latency rather than queue latency. It is also why the thread exists:
the wait is a blocking poll and cannot sit on the simulation thread.

The mean is computed over the samples as signed values and rounded away from zero on a tie —
a detail that matters only in that it makes the estimator unbiased where truncation would
drag it toward zero.

After convergence the same estimator keeps running from the receive path, fed by the game
layer whenever it receives a message carrying the server's clock.

## `timeServer` / `timeServer_Correct` / `timeServer_UserDelta`

**Contract** — `timeServer` is the client's estimate of the server's clock: its own clock plus
the smoothed offset plus the operator bias. `timeServer_Correct` feeds one more sample from a
server timestamp the game layer received, deriving the round trip from the link's reported
ping rather than from a probe. `timeServer_UserDelta` sets the bias, which is added raw and
never smoothed.

## `UpdateStatistic` / `ClearStatistic` / `GetServerAddress`

**Contract** — `UpdateStatistic` asks the transport for the link report and folds it into the
statistics record. `ClearStatistic` resets it. `GetServerAddress` resolves the connected
server's host name to an address and port, for the benefit of anything that needs to name the
server — a browser entry, a reconnect. The original resolves the name twice, forward and back,
which normalizes an alias to its canonical address; that round trip is the only reason the
second lookup exists.

## Notes

`HOST_NODE`, the per-discovered-session record, is described in the source as deprecated and
survives only because the connect path still uses the first entry to reach the server address.
A rebuild connecting to a known address needs no list at all.
