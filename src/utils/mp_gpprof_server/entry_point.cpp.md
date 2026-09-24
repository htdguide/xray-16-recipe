# src/utils/mp_gpprof_server/entry_point.cpp

> The profile server's process entry — it binds a gateway socket, names the game to the account service, and accepts requests forever.

**Needs** — [`requests_processor.h`](requests_processor.h.md) · [`sake_worker.h`](sake_worker.h.md) · [`profile_request.h`](profile_request.h.md) · [`profile_printer.h`](profile_printer.h.md) · [Seam: Networking transport](../../../SYSTEM-REQUIREMENTS.md#seam-networking-transport) · [Seam: Multiplayer matchmaking and accounts](../../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)

**Used by** — nothing in this recipe; entry point or dead code.

**Tier floor** — T2: socket set-up and an accept loop.

## Purpose

The composition root of the standalone profile server. It is a **web service behind a
gateway**: it does not speak the web protocol itself and does not listen on a public port.
A web server in front of it accepts the client's connection and forwards the request over a
socket this process binds, which is why the only configuration it takes is that socket's
address and the path prefix it is mounted under.

**It reaches nothing today.** The account service it logs in to was shut down in 2014
([Seam: Multiplayer matchmaking and accounts](../../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)),
so the request pool's construction fails during login and the process exits before it
accepts anything. What survives is the service's *shape*: one path, one response format,
batched lookups behind a cache.

This file also holds the game's registration identity, as four process-wide constants, and
that is a genuine layering accident — the identity belongs with the field registry, not
with the entry point. A rebuild puts it in configuration.

## State

```text
RECORD ServerIdentity          # compiled in; belongs in configuration
  game_name    : text
  game_id      : int
  product_id   : int
  namespace_id : int

RECORD ServerConfig
  socket_address : text        # from the first argument; required
  root_path      : text        # from the second argument; defaults to a compiled-in prefix
```

**Invariants**

- **The socket address is required and has no default.** A process started without one
  binds nothing and exits, because a gateway service with no address cannot be reached.
- **The root path is the prefix every request path must begin with**, and it must agree
  with the web server's mount point. It is process-wide and read by the request pool
  through a global, which is the only global in the tool and the only thing that keeps
  this file and [`requests_processor.cpp`](requests_processor.cpp.md) from being
  independent.
- **A request is accepted into a record that the pool takes ownership of**, and a fresh
  record is prepared before the next accept. The loop must never reuse a record it has
  handed over — the pool holds it until the answer is written.

## `main`

**Contract** — initializes the gateway library, binds the socket named by the first
argument with a fixed backlog, optionally overrides the path prefix from the second,
constructs the request pool — which starts the session thread and blocks until login
succeeds or fails — and then accepts requests until the socket closes. Any failure during
set-up is reported and exits with a failure status. Never returns while the socket is
open.

```text
FUNCTION main(arguments) -> int
  TRY
    initialize_gateway()
    socket <- bind(arguments[1], backlog = 128)
    FAIL WITH cannot_bind IF socket IS none
    IF arguments HAS a second entry THEN root_path <- arguments[2]

    pool <- new RequestPool          # starts the session thread; blocks on login
    request <- new AcceptedRequest(socket)

    WHILE accept(request) SUCCEEDS
      pool.add_request(request)      # ownership transfers; `request` is now empty
      request <- new AcceptedRequest(socket)
    RETURN success
  CATCH any failure
    report(it)
    RETURN failure
```

**Invariants**

- **Ownership of an accepted request transfers to the pool and the loop must observe that
  it did.** The original asserts the handover left its own handle empty; the decision is
  that there is exactly one owner of a connection at any moment, and the owner is whoever
  will answer it.
- The backlog is a fixed 128. It bounds how many requests the operating system holds while
  the accept loop is busy, and the accept loop is busy whenever the pump holds the session
  worker's lock — see [`requests_processor.cpp`](requests_processor.cpp.md). On a
  correctly threaded rebuild the backlog stops mattering.
- Every failure path, including a failure to log in, exits the process rather than serving
  degraded. A profile server that cannot reach the profile store has nothing to say.

**Notes**

- The accept loop is single-threaded and the pool's queueing is what makes that acceptable:
  accepting is cheap, answering is deferred. A cache hit is answered *on this thread*
  though, which means a burst of cached lookups is served serially.
- The socket address is passed through to the gateway library verbatim, so whether it names
  a port or a filesystem socket is that library's business, not this tool's.
