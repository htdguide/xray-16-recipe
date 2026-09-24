# src/xrNetServer/empty/NET_Server.cpp

> The server with every transport call removed: the roster, the bans and the filter still
> work, the session never opens.

**Needs** — [`NET_Server.h`](NET_Server.h.md) · [`../NET_Common.h`](../NET_Common.h.md) · [`../NET_Messages.h`](../NET_Messages.h.md) · [`../NET_Log.h`](../NET_Log.h.md) · [`../NET_PlayersMonitor.h`](../NET_PlayersMonitor.h.md) · [`../ip_filter.h`](../ip_filter.h.md) · [Seam: Networking transport](../../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1, inherited.

## Purpose

This is [`../NET_Server.cpp`](../NET_Server.cpp.md) with the vendor library subtracted. Read
that page for every contract; this one records the difference, which is the seam boundary
drawn by subtraction.

## State

Identical to the real server minus the two transport handles: the roster, the ban list, the
subnet filter, the server statistics, the port and the option string are all present and all
maintained. Nothing is added.

## What still works

Everything that is not the network, and it is most of the file: the client roster, the ban
list with its persistence and its expiry sweep, the subnet filter, the address record, the
per-link statistics accumulator, the envelope accumulator per client, the message dispatch and
the relay path, and the governor's rate brake.

## What was removed, and what it tells you

| Removed | What a transport must therefore provide |
|---|---|
| Endpoint creation, event callback, hosting | Bind, advertise a session, accept, deliver events |
| Discovery response | Answer a probe with the session description |
| Admission vetting hook | Ask before admitting, carry a refusal reason back |
| Per-client send | Deliver a byte span to one client on a channel |
| Eject | Server-initiated disconnect carrying a reason |
| Client address query | The peer's address and port |
| Link report query | Ping, throughput, loss, queue depth |

## `Connect` — hosting

**Contract** — parses the same option string and enters the same port-retry loop with the call
that would have bound the port removed, so it walks the range and reports failure. The ban
list and the subnet filter are still loaded first, which means a null server reads and
rewrites its configuration files on a start it then abandons.

**Notes** — the range clamp on an explicitly given port was lost in the copy. The real server
clamps a parsed port into the valid range; this one does not, so an out-of-range value is used
as given. Nothing downstream would tolerate it, but nothing checks.

## `HasBandwidth` — the server governor

**Contract** — the rate brake is intact. The depth brake is not, and it is broken rather than
absent: the queue-depth variable and the result variable that guards it were both left
declared and never assigned, so the brake tests two values that were never written.

This is the single most important line in the directory, because of what it demonstrates: the
depth brake is the part of the governor that actually protects a congested link, and a
subtraction that only meant to remove a vendor call silently removed the protection *and*
introduced a read of an unassigned value. It is unreachable — no client ever connects — and it
is exactly what happens when a seam is filled by copying instead of by implementing.

## `GetClientAddress`

**Contract** — reports success and leaves the address untouched. Its caller in the
disconnect-by-address path then compares that untouched address against the one it is looking
for, so on this filling that path's behaviour depends on whatever the address happened to
contain. Unreachable for the same reason.

## `net_Handler` / `_Recieve` / `SendTo_LL`

**Contract** — `net_Handler` is an empty acknowledgement: no admission vetting, no discovery
answer, no client creation, no probe bounce. `_Recieve` is unchanged and still performs the
oversize check and the relay. `SendTo_LL` keeps the packet trace and drops the bytes.

**Notes** — because `net_Handler` never creates a client, the roster on this filling is only
ever populated through the direct-connect path, which is how single-player works: the server
creates its own local client itself and the transport is never involved.

## Everything else

The address record, the ban entry, the statistics implementation, the roster operations, the
broadcast, the eviction-by-address walk and the configuration persistence are all present and
identical in substance. Their contracts are in [`../NET_Server.cpp`](../NET_Server.cpp.md).

The per-link statistics still exist and still roll over once a second; they simply have no
source, so every field that comes from the transport stays zero and the two byte counters the
engine maintains itself are the only live ones.
