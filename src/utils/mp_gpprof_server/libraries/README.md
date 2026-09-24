# src/utils/mp_gpprof_server/libraries — vendored third-party code, excluded

**Nothing in this directory is recipe'd, and that is deliberate.**

The original vendors three complete third-party source trees here, unmodified, so that the
profile server could be built without the host system providing them:

- a **web-gateway library** — the protocol by which a web server hands an accepted request
  to a separate long-running process, plus its stream adapters;
- the **vendor's online-services client** — account login, persistent record storage,
  availability checks, and the transport, cryptography and serialization underneath them;
- a **threading library** — an implementation of the portable thread interface for a
  platform that did not ship one.

A recipe replaces *this repository's* reasoning with prose. Vendored dependencies are not
this repository's reasoning: they are a copy of somebody else's, checked in to make a build
reproducible. Recipe'ing them would describe the wrong system, and at great length — the
three trees together are an order of magnitude larger than everything in this chapter that
was actually written here.

What a rebuilder needs from them is stated as seams and contracts elsewhere:

- The gateway protocol is a shopped dependency; what the server does with it — bind a
  socket, accept, answer, finish — is on
  [`entry_point.cpp`](../entry_point.cpp.md) and
  [`profile_request.cpp`](../profile_request.cpp.md).
- The online-services client is
  [Seam: Multiplayer matchmaking and accounts](../../../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts),
  marked *given, and dead*: the service stopped answering in 2014 and there is no
  compatible substitute. The shape of the exchange, as far as it constrains a replacement,
  is on [`gamespy_sake.cpp`](../gamespy_sake.cpp.md).
- The threading library is
  [Seam: Threads, atomics and process services](../../../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services);
  the four primitives this tool actually uses are on [`threads.h`](../threads.h.md).

A rebuild vendors nothing here. It takes a gateway library, or speaks the web protocol
directly; it uses its own language's threads; and it either drops the account service
entirely or designs a fresh one.
