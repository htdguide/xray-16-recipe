# src/xrNetServer/NET_Shared.h

> The module's shared vocabulary: the tunables that pace both ends, the debug-flag set, and
> the per-link statistics record the bandwidth governor reads.

**Needs** — [`xrCore/client_id.h`](../xrCore/client_id.h.md) · [`xrCore/FTimer.h`](../xrCore/FTimer.h.md) · [`xrCore/_flags.h`](../xrCore/_flags.h.md) · [`NET_Compressor.h`](NET_Compressor.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — [`NET_AuthCheck.h`](NET_AuthCheck.h.md) · [`NET_Client.h`](NET_Client.h.md) · [`NET_Common.h`](NET_Common.h.md) · [`NET_Log.h`](NET_Log.h.md) · [`NET_PlayersMonitor.h`](NET_PlayersMonitor.h.md) · [`NET_Server.h`](NET_Server.h.md) · [`NET_Client.h`](empty/NET_Client.h.md) · [`NET_Server.h`](empty/NET_Server.h.md) · [`stdafx.h`](stdafx.h.md) · [`PHNetState.cpp`](../xrServerEntities/PHNetState.cpp.md) · [`script_net_packet_script.cpp`](../xrServerEntities/script_net_packet_script.cpp.md)
**Tier floor** — T2: it is a settings record and an interface. What keeps it off T3 is the
per-link statistics object, which is read every frame by the governor and must not cost an
allocation to consult.

## Purpose

Everything in this module needs the same handful of settings — how often to send, how deep a
backlog to tolerate, which traffic to trace — and the client and the server both need the same
per-link statistics surface. This file is where those live so neither end has to depend on the
other.

The statistics type is declared here but implemented in [`NET_Server.cpp`](NET_Server.cpp.md),
which is arbitrary: it landed there because the server was written first. A rebuild should put
it with the transport adapter, since every field in it comes from the transport.

## State

```text
# Process-wide settings. All are console variables; the values are defaults.
RECORD NetworkSettings
  client_update_rate   : int   = 30    # world updates per second the client may send
  client_pending_limit : int   = 2     # outbound datagrams in flight before the client backs off
  server_update_rate   : int   = 30    # world updates per second the server may send per client
  server_pending_limit : int   = 3     # outbound datagrams in flight before the server backs off
  player_name          : text  = "Player"   # at most 31 characters plus terminator
  direct_connect       : bool  = false      # client and server share this process
  debug_flags          : set<DebugFlag>

ENUM DebugFlag
  minimize_updates     # collapse both update rates to one per second
  dump_packet_sizes    # report the size of each outbound message
  log_server_packets   # trace every server-side packet to a file
  log_client_packets   # trace every client-side packet to a file

# The broadcast destination: a reserved client identifier meaning "everyone".
CONSTANT broadcast_client = 0xFFFFFFFF
```

**Invariants** — `direct_connect` is set when the server is started with the single-player
marker in its option string, and cleared whenever either end shuts down. While it is set, the
governor keeps its rate brake but drops its backlog brake, and the compressor is bypassed:
there is no network, so there is no queue to watch and nothing to be gained by shrinking a
buffer that never leaves the process.

The server's pending limit is 3 and the client's is 2. The asymmetry is deliberate in
direction if not in magnitude — the server is talking to many peers and can afford a little
more slack per peer than a client with one link — but the specific numbers are not derived
anywhere.

## `IClientStatistic`

**Contract** — one per link, on both ends. It is a thin accumulator over what the transport
reports, plus two byte counters the engine maintains itself because the transport counts
*datagrams* and the engine wants *bytes*. Consulting any field is a field read; refreshing it
asks the transport, which may block briefly, and so happens at most once per governor
decision.

```text
RECORD LinkStatistics
  # Straight from the transport, refreshed on demand
  ping_ms            : int
  throughput_bps     : int
  peak_bps           : int
  dropped            : int     # datagrams the transport gave up on
  retried            : int     # datagrams the transport resent
  # Derived, recomputed at most once a second
  messages_in_rate   : int     # per second
  messages_out_rate  : int     # per second
  bytes_in_rate      : int     # per second
  bytes_out_rate     : int     # per second
  # Accumulators the engine feeds directly at each send and receive
  bytes_out          : int
  bytes_in           : int
  times_blocked      : int     # governor refusals caused by a full outbound queue

FUNCTION refresh(from : TransportLinkReport)
  IF at least one second has elapsed since the last roll-over THEN
    roll the four accumulators into the four rates and zero them
  retain the transport's report for the direct read-outs
```

**Notes** — the roll-over threshold is "999 milliseconds or more", not 1000; with a
millisecond clock that makes the window inclusive rather than exclusive, and the rates are
therefore "per second" only approximately. Nothing depends on the precision.

`times_blocked` is the only field the governor *writes*. It exists so that a client being
starved by a congested link is distinguishable in the statistics from a client that simply has
nothing to say — which is otherwise invisible, since both look like an absence of traffic.

The record is copyable, which is unusual for something holding a transport report, and the
source says outright that the copy exists to work around a defect in a caller that takes one
by value. A rebuild should not reproduce the copy; it should fix the caller.

Clearing the statistics preserves the timer reference and the roll-over base and zeroes
everything else. The original does this by blanking the whole record and restoring two fields
afterwards — a mechanism, not a decision; a rebuild should simply construct a fresh record
bound to the same clock.

## `TimeGlobal` / `TimerAsync`

**Contract** — both return the elapsed milliseconds of the supplied monotonic clock, and both
are the *same* function. The two names record a distinction that no longer exists: one was
once safe to call from a transport callback thread and the other was not. A rebuild has one
clock read.

## Notes

Two accessor functions for the update rates are declared here and are defined nowhere and
called nowhere. They are dead declarations; a rebuild drops them.

The module's export/import decoration on every declaration is an artifact of building this as
a separately-loadable module. What survives is the fact that these settings are process-wide
and shared by two modules that do not otherwise know each other, not the mechanism by which
the symbols are shared.
