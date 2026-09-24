# src/xrGame/gamespy — what the game tells a server directory

Part of chapter 26 of [`SYSTEM-REQUIREMENTS.md`](../../../SYSTEM-REQUIREMENTS.md#7-build-order),
alongside [`ik`](../ik/README.md) and [`CdkeyDecode`](../CdkeyDecode/README.md), with
which it shares nothing but a chapter number.

Two files. They are the only part of the matchmaking integration that lives inside the
game module, and the reason they do is that the questions they answer are questions about
a *match* — which mode, which map, whose score, what the friendly-fire multiplier is —
and only the game module can answer those. Everything else in the integration is in
[`src/xrGameSpy`](../../xrGameSpy/README.md), the module that owns the vendor SDK.

The service these files talk to was shut down in 2014. The code paths still run; they
reach nothing. See
[Seam: Multiplayer matchmaking and accounts](../../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts),
whose verdict is *given, and dead*: a rebuild should carry this as an optional module with
a null implementation, and design a fresh master-server protocol if it wants working
multiplayer.

**What is worth taking from here is not the glue but the schema.** These two files are the
shipped answer to "what does a player see in a server browser, and what does a server have
to be able to report about itself". That list survives any change of directory service,
and it is short enough to read in one sitting.

## Where it sits

It rests on the running server, the current game mode, and the directory endpoint the
server holds. It has no state of its own and nothing depends on it — remove it and the
engine still runs matches, it simply does not appear in anybody's list.

## Load-bearing ideas, named once

**One property per call, answered synchronously.** The directory does not receive a
snapshot; it asks for one named property at a time and the engine reads it off the live
server there and then. There is no copy, no lock and no consistency guarantee across the
properties of one reply — two fields of the same response can come from two different
simulation steps. Nothing in the shipped behaviour cares, but a rebuild that wants a
coherent row must snapshot at the start of a query.

**The advertised schema is the union over every game mode, and never varies.** A server
declares, once, that it reports all thirty-seven match properties, and then answers the
ones that belong to modes it is not running with an empty string. This is what lets a
browser lay out one fixed table of columns. The properties are grouped by the mode that
owns them: base match facts, then the multiplayer base, then deathmatch, then team
deathmatch, then artefact hunt — each a specialization of the one before.

**A query must never be able to fail the match.** Every callback recovers the server from
an opaque handle and gives up quietly if anything is missing, and an unknown property
answers empty rather than asserting. The single exception is the team-count callback,
which asserts on an unknown game mode — deliberately, because that would mean the mode
table and the reporting switch had diverged.

**A dedicated server is its own client.** It occupies the first client slot, so every
player index the directory asks about is shifted by one before the client list is walked.
Forgetting this makes a dedicated server report a phantom player and hide the last real
one.

**There is no stable player identity in the reply.** Players are reported by position in
the server's client collection. When someone leaves, everyone after them moves up a row.

**Address negotiation is registered for and not implemented.** A client behind address
translation can find the server in a list and cannot reach it. This is the largest gap in
the integration and the first thing a fresh master-server design should fix.

**Banned addresses are refused at the query channel, not just at join.** The one callback
here with a security purpose.

## The twins

| File | Role |
|---|---|
| [`GameSpy_QR2_callbacks.h`](GameSpy_QR2_callbacks.h.md) | The eight questions a directory service can ask — worth reading as the interface a replacement must satisfy. |
| [`GameSpy_QR2_callbacks.cpp`](GameSpy_QR2_callbacks.cpp.md) | The answers: the advertised property schema, the per-player and per-team rows, the counts, and the failure rules. |
