# src/xrGame/file_transfer.cpp

> The two ends of in-band file transfer: the server, which runs many transfers keyed by destination and source, and the client, which sends at most one at a time.

**Needs** — [`file_transfer.h`](file_transfer.h.md) · [`filetransfer_node.h`](filetransfer_node.h.md) · [`filereceiver_node.h`](filereceiver_node.h.md) · [`filetransfer_common.h`](filetransfer_common.h.md) · [`Level.h`](Level.h.md) · [`xrServer.h`](xrServer.h.md) · [`xrServerEntities/xrMessages.h`](../xrServerEntities/xrMessages.h.md) · [`xrNetServer/NET_Messages.h`](../xrNetServer/NET_Messages.h.md) · [`xrCore/buffer_vector.h`](../xrCore/buffer_vector.h.md) · [`xrUICore/ui_base.h`](../xrUICore/ui_base.h.md) · [`xrEngine/StatGraph.h`](../xrEngine/StatGraph.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport) · [Seam: Debug overlay UI](../../SYSTEM-REQUIREMENTS.md#seam-debug-overlay-ui)

**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: session bookkeeping, timeouts and message dispatch over the transport

## Purpose

Multiplayer moves whole files over the game connection: a map or a skin a joining client
lacks, a screenshot, a voted-on demo. This file owns the *sessions* — who is sending what to
whom, who is receiving, what happens when a peer disappears mid-transfer, and when a stalled
transfer is given up on. The per-session mechanics are in
[`filetransfer_node.cpp`](filetransfer_node.cpp.md) and
[`filereceiver_node.cpp`](filereceiver_node.cpp.md).

The asymmetry between the two sites is the design:

- **The server** may be sending to many clients at once and receiving from many at once. A
  send is keyed by *(destination, source)* — a pair, not one identifier — because the server
  also relays, forwarding a file from one client to another while still naming the original
  sender. Receives are keyed by sender alone.
- **The client** may be sending exactly one file, and always to the server. Receives are
  still keyed by sender, because the server names whose file it is forwarding.

## State

```text
RECORD ServerSite
  transfers : map<(destination_peer, source_peer), FileTransferNode>
  receivers : map<source_peer, FileReceiverNode>

RECORD ClientSite
  transfering : optional<FileTransferNode>          # at most one
  receivers   : map<source_peer, FileReceiverNode>
```

Invariants:

- **One send per (destination, source) pair on the server; one send in total on the client.**
  Both are enforced by refusing a second start with a logged error rather than by queuing.
  A caller that wants two transfers to one peer must serialise them itself.
- **One receive per source peer, both sides.** Same enforcement.
- **A session removed from either map is destroyed immediately**, and destroying an
  *incomplete* session tells the peer: an incomplete send emits an abort, an incomplete
  receive emits a rejection. That pairing is what keeps the other end from waiting out a
  full timeout.

## `update_transfer` (server)

**Contract** — called once per server update. For each active send: verify the destination
peer still exists, adjust the chunk size from that peer's throughput statistics, compose and
send one chunk, and report progress. Completed and orphaned sessions are collected and torn
down *after* the walk.

```text
FUNCTION update_transfer()
  IF transfers is empty: RETURN
  to_stop = empty list
  FOR EACH ((destination, source), node) IN transfers
    peer = server.client_by_id(destination)
    IF peer is none                               # the client left mid-transfer
      node.report(sending_rejected_by_peer)
      to_stop.append(key) ; CONTINUE
    IF NOT node.is_ready_to_send(): CONTINUE

    node.calculate_chunk_size(peer.stats.peak_bps, peer.stats.bps)
    packet = new message(FILE_TRANSFER)
    packet.write_byte(receive_data)
    packet.write_int32(source)                    # whose file this is
    complete = node.make_data_packet(packet)
    server.send_to(destination, packet, reliable+ordered)
    node.report(complete ? sending_complete : sending_data)
    IF complete: to_stop.append(key)
  FOR EACH key IN to_stop: stop_transfer_file(key)
```

**Invariants** — sessions are never erased while the map is being walked; the keys are
collected into a scratch list and the teardown happens afterwards. The teardown calls
progress callbacks, which may start new transfers, so doing it inside the walk would
invalidate the iteration twice over.

**Notes** — the scratch list is stack-allocated at exactly the map's current size, which is
a safe upper bound because the walk can add at most one key per entry. That is an allocation
decision, not a design one; what survives is *collect then destroy*.

A departed peer is reported to the sender as "rejected by peer" rather than as a distinct
"peer gone" status. The vocabulary has no such value, so the two are indistinguishable to a
progress callback. Worth knowing: a caller cannot tell a refusal from a disconnection.

One chunk per session per update. With many active transfers the server sends one chunk to
each every update, so the per-client rate is the chunk size times the update rate and the
*aggregate* scales linearly with the number of clients. There is no server-wide budget. A
rebuild serving many clients over one uplink will need one.

## `update_transfer` (client)

**Contract** — the same shape for the single outbound transfer, using the client's own
connection statistics, sending to the server implicitly (no destination is named). Also
drives the debug throughput graph.

**Notes** — the client's outbound message carries **no source identifier**, while the
server's carries one. The two commands are otherwise identical. That is the protocol's one
genuine asymmetry and it is why the two sites cannot share a message reader: the server
reads a client's message without a source field (it knows the sender from the connection),
and the client reads the server's with one (it needs to know whose file is arriving).

## `on_message` (server)

**Contract** — dispatches one inbound file-transfer message from a named sender on the three
commands.

```text
FUNCTION on_message(packet, sender)
  SWITCH packet.read_byte()
    receive_data:
      session = receivers[sender]
      IF none
        send reject(peer 0) to sender          # I did not ask for this
        RETURN
      IF session.receive_packet(packet)
        session.report(receiving_complete) ; stop_receive_file(sender)
      ELSE
        session.report(receiving_data)

    abort_receive:                              # the sender gave up
      session = receivers[sender]
      IF session exists
        session.report(receiving_aborted_by_peer) ; stop_receive_file(sender)

    receive_rejected:                           # the destination refused my send
      key = (sender, peer named in the packet)
      session = transfers[key]
      IF session exists
        session.report(sending_rejected_by_peer) ; stop_transfer_file(key)
```

**Invariants** — unsolicited data is answered with a rejection, every time. That is what
stops a peer from streaming bytes at a server that never asked, and it is also how a
receiver that has already torn down its session tells the sender to stop.

**Notes** — the rejection sent for unsolicited data names peer zero rather than the actual
sender, because the client side ignores that field on a rejection anyway. Sloppy but
consistent with the message constructors.

The rejection case is the only one that reads a peer identifier out of the message, and it
needs it because the server's send key is a *pair*: a client refusing a file must say whose
file it is refusing, since the server may be relaying two different files to it.

## `on_message` (client)

**Contract** — the same three commands, but the source peer is read from the message first,
since the client must know whose file the server is forwarding. The rejection case ignores
that identifier, because the client has only one outbound transfer.

**Notes** — unknown abort and unknown rejection messages are logged as warnings and
otherwise ignored, where the server silently ignores the equivalent. Neither is a protocol
error: both happen naturally when both ends tear down at once.

## `stop_obsolete_receivers`

**Contract** — both sites, identical code. Walks the receive sessions and reaps any that
have gone quiet, with **two different timeouts** depending on whether anything has arrived
yet. Reports a timeout status before tearing down. Collect-then-destroy, like the update.

```text
FUNCTION stop_obsolete_receivers()
  FOR EACH (source, session) IN receivers
    IF session.downloaded_size == 0               # nothing has arrived yet
      IF session.last_read_time == 0
        session.last_read_time = global_clock     # arm the clock on first sight
      ELSE IF global_clock - session.last_read_time > 28000 ms
        session.report(receiving_timeout) ; reap
    ELSE                                          # a transfer in progress
      IF global_clock - session.last_read_time > 6000 ms
        session.report(receiving_timeout) ; reap
```

**Notes** — the two timeouts encode two different failures. Before the first chunk, the
session is waiting for a peer that may not have started sending yet, so it waits about
twenty-eight seconds — the source describes it as fourteen maximum pings of two seconds
each. Once data is flowing, a six-second gap (three maximum pings) means the peer has
stopped, and waiting longer buys nothing. The generous start and the tight middle are the
right shape; the specific ping figure of two seconds is an assumption about the worst
tolerable connection and is the only magic number here.

Arming the clock lazily — on the first sweep that sees the session rather than at creation —
means the start timeout is measured from the first *sweep* after the session opened, not
from the session's creation. Off by up to one update; immaterial against twenty-eight
seconds, and a rebuild should just stamp the creation time.

## `start_transfer_file`

**Contract** — four spellings on the server (file, memory block, buffer list, streaming
memory writer) and two on the client (file, memory block). All refuse when a transfer for
that key is already active, logging an error and doing nothing. The file spelling can also
fail at open, and tears the session straight back down if it does.

**Notes** — the initial chunk size differs by site, not by source: the **server starts every
transfer at the maximum chunk size and the client at the minimum**. Together with the rate
controller's server-pins-at-maximum rule
([`filetransfer_node.cpp`](filetransfer_node.cpp.md)), the policy is: the server pushes at
full rate immediately, and a client trickles and slowly ramps. That is right for the shipped
topology — a server's uplink serves everyone and a client's is shared with its own gameplay
traffic — and it is the only place the two sites' behaviour genuinely differs in kind.

The failed-open path creates the session and *then* tears it down rather than checking
first. Incidental, but note the consequence: the progress callback sees the session's
teardown, including the abort message that goes out to the peer, for a transfer that never
began.

## `stop_transfer_file` / `stop_receive_file`

**Contract** — remove and destroy a session. **If it was incomplete, tell the peer first**: a
send emits an abort naming the source, a receive emits a rejection. A send to a peer that is
already gone skips the message. Missing sessions are logged and ignored.

**Invariants** — this is the *only* place the two teardown messages are emitted, and routing
every teardown through it is what guarantees the other end always learns. Every path — user
cancellation, completion, timeout, peer loss, destruction of the whole site — funnels here.

**Notes** — a *complete* session is torn down silently. That is the difference between
"finished" and "given up on", and it is the reason the completeness test exists on both node
types.

Destruction of either site drains both maps by repeatedly tearing down the first entry
rather than iterating, because each teardown mutates the map. A rebuild with stable
iteration can walk it; the repeated-first-entry shape is a consequence of the container, not
a decision.

## `is_transfer_active` / `is_receiving_active`

**Contract** — presence tests on the two maps. The client's transfer test is simply whether
the single slot is occupied.

## Debug throughput graph

**Contract** — debug builds only, client side only. Draws the chunk size over time against
the two bounds, created lazily when the statistics overlay is switched on and destroyed when
it is switched off.

**Notes** — it plots the chunk size, not the measured throughput, which is the honest thing
to plot: the chunk size *is* what this code controls, and watching it saw between the bounds
is how the random re-probe policy in the rate controller was meant to be observed.
