# src/utils/mp_gpprof_server/profile_request.h

> Declares one in-flight web request — the player name extracted from its path, and the two ways it can be answered.

**Needs** — [`profile_data_types.h`](profile_data_types.h.md) · [Seam: Networking transport](../../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)

**Used by** — [`entry_point.cpp`](entry_point.cpp.md) · [`profile_request.cpp`](profile_request.cpp.md) · [`profiles_cache.h`](profiles_cache.h.md) · [`requests_processor.cpp`](requests_processor.cpp.md) · [`requests_processor.h`](requests_processor.h.md)

**Tier floor** — T2: it owns a connection that must be released at a defined moment.

## Purpose

Declares the surface implemented in [`profile_request.cpp`](profile_request.cpp.md). One
record and one free function, and the split between them is the decision: **the name is
extracted from the path before the request object exists**, because a path that names no
player is refused without ever becoming a pending request.

## Exported units

- `extract_username` — pull a player name out of a request path, undoing its escaping.
- `fetch_profile_request` — one accepted request awaiting an answer. Owns the connection
  from acceptance until it is completed, and completing it is the only way to release it.
- `get_profile_name` — the name the request asked about.
- `complete_success` — answer with the profile and close.
- `complete_failed` — answer "no such profile" and close.
- `request_uri_t`, `user_name_t` — the two fixed-width text buffers the tool passes around.
  Their widths are a property of the original's memory discipline, not of the protocol; a
  rebuild uses growable text and must then bound the name's length itself, because the
  cache in [`profiles_cache.h`](profiles_cache.h.md) stores names inline.

**Notes**

- There is no third completion. A request that is neither answered nor refused holds its
  connection open until the process exits, and nothing in the tool times one out.
