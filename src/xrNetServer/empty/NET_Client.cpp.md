# src/xrNetServer/empty/NET_Client.cpp

> The client with every transport call removed: the session bookkeeping still runs, the
> connection never forms.

**Needs** — [`NET_Client.h`](NET_Client.h.md) · [`../NET_Common.h`](../NET_Common.h.md) · [`../NET_Messages.h`](../NET_Messages.h.md) · [`../NET_Log.h`](../NET_Log.h.md) · [`../NET_Server.h`](../NET_Server.h.md) · [Seam: Networking transport](../../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — reached through its declarations in [`NET_Client.h`](NET_Client.h.md); callers name that, not this file.
**Tier floor** — T1, inherited: it still reinterprets received bytes as system-packet memory
images, even though none ever arrive.

## Purpose

This is [`../NET_Client.cpp`](../NET_Client.cpp.md) with the vendor library subtracted. Read
that page for every contract; this one records only what the subtraction changed, because the
difference *is* the seam boundary — everything still here is engine logic, everything missing
is what a substitute transport must supply.

A rebuild does not carry this file. It carries the contract in
[`../README.md`](../README.md) and one honest null filling of it.

## State

Identical to the real client, minus the transport handles: connection state, the message
queue, the offset ring, the statistics record and the governor's bookkeeping are all present
and all maintained.

## What still works

The message queue, the envelope accumulator, the option-string parse, the governor's rate
brake, the packet trace, and the classifier that separates system packets from engine
messages. All of it would function the moment bytes arrived.

## What was removed, and what it tells you

| Removed | What a transport must therefore provide |
|---|---|
| Endpoint creation and its event callback | A connection object and an event stream |
| Discovery | Probe a host and receive its session description |
| Connect, both the local and the remote path | Establish a connection, report refusal reasons distinguishably |
| Send | Deliver a byte span on a channel |
| Link report query | Ping, throughput, loss, queue depth |
| Server address query | The peer's address and port |

## `Connect`

**Contract** — parses the option string exactly as the real client does, then enters the
port-retry loop with the call that would have bound the port deleted. The loop's condition
therefore never becomes true: it walks to the end of the port window and reports failure. A
caller sees an ordinary "could not connect".

**Notes** — the reported reason is "every port is busy", which is a lie in an informative
direction only by accident. An honest null filling says "no transport on this platform" and
returns immediately.

The direct-connect path is untouched and still returns success, which is exactly right: in
that mode there is no transport to be missing, and single-player works on this filling.

## `_Recieve`

**Contract** — the same classifier as the real client, with one defect. In the real client the
engine-message branch is the *alternative* to the system-packet branch; here it is a separate
test, and the unrecognized-system-packet case has lost its early return. A packet that carries
the magic prefix but matches neither known length is reported as unknown **and then handed to
the engine as a message**.

Unreachable today, since nothing delivers packets to this filling. It is recorded because it
is precisely the class of divergence that makes two parallel copies the wrong structure.

## `Sync_Thread` / `Sync_Average`

**Contract** — both are empty. The synchronization thread is still spawned and exits
immediately, so a client on this filling never becomes synchronized. Nothing notices, because
nothing connects.

## `UpdateStatistic` / `SendTo_LL` / `GetServerAddress`

**Contract** — `UpdateStatistic` does nothing. `SendTo_LL` keeps the byte accounting and the
trace and drops the bytes. `GetServerAddress` reports success and fills in nothing, which is
the one removal that is actively misleading: the caller cannot tell the difference between a
resolved address and an unfilled one.

## `INetQueue`

**Contract** — the same queue, with the free-list decay rule removed: buffers are recycled but
never released, so the free list only grows. Since each buffer is 16 kilobytes, a traffic
burst on this filling would pin its peak footprint for the life of the process. Another
drift, in the direction of a leak.

The explicit lock bracket gained proper names here (`LockQ`/`UnlockQ` against the real one's
`Lock`/`Unlock`), which is the one place the null copy reads better than the original.
