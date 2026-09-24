# src/utils/mp_gpprof_server — the standalone profile server

Part of chapter 28 of [the build order](../../../SYSTEM-REQUIREMENTS.md#7-build-order).

## What this module is responsible for

Serving one player's multiplayer awards and best streaks over the web, as plain text, from
a remote account service. It exists so that a community website could show a player's
record without embedding the vendor's client library.

**It reaches nothing today.** The vendor service it logs in to was shut down in 2014
([Seam: Multiplayer matchmaking and accounts](../../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)),
so the process now fails during start-up, at the availability check, before it accepts a
single request. Nothing in the game depends on it and nothing in a rebuild has to work.

It is recipe'd for two things that outlive the service: **the data model** — what a
player's multiplayer history consists of — and **the request and response shapes** the
service exposes. A rebuilder designing a replacement reads this chapter for what to serve.
Per the seam, this recipe documents what the game asks for, not how the dead service
answered, so the vendor's own call sequence is described only as far as it reveals the
protocol's shape, and its library is not described at all.

## Where it sits

Nowhere in the engine. It shares no code with it — not even the core layer — and links
only its own copy of the data model, a web-gateway library, a threading library and the
vendor's client. It is a separate program, built separately, run beside a web server.

## Load-bearing ideas, named once

**The service is one path and one response.** A request for `<root>/<player name>` is
answered with that player's profile as plain text; anything else is answered "not found".
The root prefix is configurable and must match the mount point of the web server in front.

**The response body is sixty-seven lines, always.** A blank line, the player's name as
requested, then each award's count and last-earned date under its published name — the
date under the same name with a fixed suffix — then each best streak. Every line is
present, zero-valued when nothing was earned, so a consumer can parse positionally and
never handle an absent field.

**The data model is thirty awards and seven streaks, index-addressed.** An award's identity
is its position in the enumeration; the published name is presentation. A count is half as
wide as a date. A whole profile is a flat block of counters with no allocation, which is
what lets the cache size itself in records. The date's epoch and encoding are **not
recoverable from this repository** — it is fetched, cached and printed as an integer, end
to end.

**Profiles are fetched in batches, never one at a time.** A remote query costs a round trip
whether it asks about one player or a hundred, and requests arrive one at a time. So the
whole architecture exists to make batches: requests accumulate in a queue, a pump running
on the session's thread sweeps up everything waiting, asks once, and answers all of them
together. A request's latency is therefore the pump's poll interval plus a round trip.

**A cache hit short-circuits the queue entirely**, answered on the accepting thread. The
cache is a fixed-capacity sorted array with two policies: entries expire by age, and when
it is full the sweep drops every entry that has never been served. The cache is what keeps
the queue short.

**The remote session is single-threaded and owned by one thread for the life of the
process.** Everything else reaches it by submitting a task. It only advances inside an
explicit time slice — nothing happens between calls — which is why the pump is a polling
loop rather than an event wait, and why a rebuild with a normally asynchronous client
deletes the pump outright.

**The service account's credentials are compiled in, in plain text.** So is the record
store's shared secret. A rebuild takes them from configuration.

**The query filter is built by string concatenation into a query language**, and the only
defence against injection is dropping any player name containing a quote — done at the
point of use, far from where the name enters. A rebuild rejects at the door and
parameterizes.

## Known defects worth not reproducing

The implementation carries several errors that a rebuilder should know about rather than
copy, each detailed on its own page:

- The monotonic clock measures processor time at one-second resolution, so cache expiry
  effectively never fires ([`threads.h`](threads.h.md)).
- Sub-second sleeps are a thousand times shorter than asked, so the pump spins
  ([`threads.h`](threads.h.md)).
- Every queued task runs while the worker's queue lock is held, so the accepting thread
  stalls behind the fetching thread ([`sake_worker.cpp`](sake_worker.cpp.md)).
- A batch whose names are all rejected leaves a query permanently outstanding — a hang
  reachable from a single request ([`gamespy_sake.cpp`](gamespy_sake.cpp.md)).
- The player-name column is matched by inequality against the wrong name space; two errors
  that cancel ([`gamespy_sake.cpp`](gamespy_sake.cpp.md)).
- The cache eviction sweep's condition is nearly inert
  ([`requests_processor.cpp`](requests_processor.cpp.md)).

## The files

| File | Role |
|---|---|
| [`profile_data_types.h`](profile_data_types.h.md) | **The data model**: the profile record, the award vocabulary, the streak vocabulary |
| [`profile_data_types.cpp`](profile_data_types.cpp.md) | The tables joining that vocabulary to the service's field numbering |
| [`profile_printer.h`](profile_printer.h.md) | **The response body format** |
| [`profile_request.h`](profile_request.h.md) · [`profile_request.cpp`](profile_request.cpp.md) | The request path shape, name extraction and decoding, and the two completions |
| [`requests_processor.h`](requests_processor.h.md) · [`requests_processor.cpp`](requests_processor.cpp.md) | The batching architecture: the two queues, the pump, the cache interaction |
| [`profiles_cache.h`](profiles_cache.h.md) · [`profiles_cache.cpp`](profiles_cache.cpp.md) | The bounded profile cache and its two eviction policies |
| [`sake_worker.h`](sake_worker.h.md) · [`sake_worker.cpp`](sake_worker.cpp.md) | The single thread owning the remote session, and its task queue |
| [`gamespy_sake.h`](gamespy_sake.h.md) · [`gamespy_sake.cpp`](gamespy_sake.cpp.md) | The remote exchange: login, the batched query, paging, row decoding |
| [`threads.h`](threads.h.md) · [`threads.cpp`](threads.cpp.md) | The tool's private lock, condition, worker and clock |
| [`atlas_stalkercoppc_v1.h`](atlas_stalkercoppc_v1.h.md) · [`atlas_stalkercoppc_v1.c`](atlas_stalkercoppc_v1.c.md) | The generated field-number registry; frozen, and meaningless without the service |
| [`entry_point.cpp`](entry_point.cpp.md) | Process entry: the gateway socket, the game identity, the accept loop |
| [`libraries/`](libraries/README.md) | Vendored third-party code, excluded from the recipe |
