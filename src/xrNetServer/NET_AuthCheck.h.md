# src/xrNetServer/NET_AuthCheck.h

> Declares the content-authentication path lists and the exemption test.

**Needs** — [`NET_Shared.h`](NET_Shared.h.md)
**Used by** — [`xrServer_Connect.cpp`](../xrGame/xrServer_Connect.cpp.md) · [`NET_AuthCheck.cpp`](NET_AuthCheck.cpp.md)
**Tier floor** — T3: two declarations over lists of text.

## Purpose

Declares the surface implemented in [`NET_AuthCheck.cpp`](NET_AuthCheck.cpp.md).

## Exported units

- **`fill_auth_check_params(ignore, check)`** — produces the two path lists: what must match
  between server and client, and what is exempt.
- **`allow_to_include_path(ignore, path)`** — whether a path may take part in the
  authentication checksum. Also consulted by the configuration parser when following include
  directives, so the authenticated set and the loaded set stay the same files.

## Notes

These are free functions rather than members of anything, and that is right: the lists are
policy, not state, and nothing owns them. A rebuild should keep them as data — a table the
build or the configuration supplies — rather than as code, since every entry is a judgement
that will differ for a different game.
