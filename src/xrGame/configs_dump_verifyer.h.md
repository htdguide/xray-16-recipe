# src/xrGame/configs_dump_verifyer.h

> Declares the server-side dump verifier implemented in [`configs_dump_verifyer.cpp`](configs_dump_verifyer.cpp.md).

**Needs** — [`configs_dump_verifyer.cpp`](configs_dump_verifyer.cpp.md) · [`mp_config_sections.h`](mp_config_sections.h.md) · [`xrCore/Crypto/xr_dsa_verifyer.h`](../xrCore/Crypto/xr_dsa_verifyer.h.md)
**Used by** — [`anticheat_dumpable_object.h`](anticheat_dumpable_object.h.md) · [`configs_dump_verifyer.cpp`](configs_dump_verifyer.cpp.md) · [`game_cl_mp.h`](game_cl_mp.h.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the surface implemented in
[`configs_dump_verifyer.cpp`](configs_dump_verifyer.cpp.md). The server owns one; it caches
its own configuration dump at construction, so constructing it is the expensive part and
verifying is cheap.

Exported units:

- **`dump_verifyer`** — the verification context, built from the shared domain parameters and
  the public key.
- **`configs_verifyer`** — with **`verify`** (the whole check, yielding a human-readable
  difference on failure), and privately **`verify_dsign`** (reconstruct the signed bytes and
  check the signature), **`get_diff`** and **`get_section_diff`** (localize a mismatch).
