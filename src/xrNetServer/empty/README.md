# src/xrNetServer/empty — the null transport filling

The transport is a [given seam](../../../SYSTEM-REQUIREMENTS.md#seam-networking-transport) and
the original fills it with a library that exists on one operating system. Every other platform
still has to build, link and run the engine, and the engine's single-player path still has to
construct a server and a client and connect them to each other.

This directory is how that is arranged: a second copy of the client and the server with every
call into the vendor library deleted and everything else left in place. It compiles
everywhere, satisfies every symbol the rest of the engine expects, and **connects to nothing**.

## What it actually does

Read the two files as the *shape of the port*, not as an implementation.

- **The session bookkeeping survives.** Connection state, the message queue, the envelope
  accumulator, the bandwidth governor's rate brake, the client roster, the ban list, the
  subnet filter, the packet trace: all present, all working, all reachable.
- **Everything that would reach the network is gone.** Creating an endpoint, advertising a
  session, discovering a host, sending, receiving, querying the link — deleted, leaving the
  surrounding control flow behind.
- **Both `connect` and `host` therefore fail.** Each retains its port-retry loop with the call
  that would have succeeded removed, so the loop walks its range and reports that every port
  is busy. This is accidental rather than designed, but the outcome is the honest one: there
  is no transport, and the attempt fails.
- **Time synchronization is a pair of empty functions**, so a client on this filling never
  becomes synchronized. Since it also never connects, nothing observes that.
- **The link statistics report nothing**, because their only source is the transport. The
  governor's depth brake therefore always passes and only the rate brake is live.

## Why the build selects it the way it does

The module's build description lists **both** copies as sources and comments out the real
ones. So the null filling is what is compiled by default on *every* platform, and the real
client and server are switched in by editing the build description. That is unusual and worth
stating plainly: the shipped default build of this module is the one that cannot connect, and
multiplayer is opted into.

## What a rebuild should do instead

Not this. Two parallel copies of the same eight hundred lines is the failure mode a seam
exists to prevent — the two have already drifted, and in three ways that are defects rather
than simplifications: the null client's message queue has lost the free-list decay rule that
keeps a traffic burst from pinning its peak memory forever; the null server's port parsing has
lost the range clamp, so an out-of-range port from the option string is used as given; and the
null client's message classifier has lost an early return, so an unrecognized system packet
falls through and is handed to the engine as a message.

The right shape is the one the rest of the recipe assumes: **one** client and **one** server,
written against the transport contract in [`../README.md`](../README.md), with the vendor
library and a do-nothing implementation as two fillings of that contract. The do-nothing
filling should report failure from `connect` and `host` directly and honestly rather than by
exhausting a port loop.

What the directory is genuinely useful for is the inventory: everything still present in these
files is engine logic that does not depend on the transport, and everything deleted from them
is the exact surface a substitute must provide. It is the seam boundary, drawn by subtraction.

| File | Role |
|---|---|
| [`NET_Client.h`](NET_Client.h.md) | The client surface with the vendor's types removed from it |
| [`NET_Client.cpp`](NET_Client.cpp.md) | The client with every transport call deleted |
| [`NET_Server.h`](NET_Server.h.md) | The server surface with the vendor's types removed from it |
| [`NET_Server.cpp`](NET_Server.cpp.md) | The server with every transport call deleted |
