# src/utils/mp_configs_verifyer/configs_dump_verifyer.h

> Declares the verifier — a signature checker bound to the frozen parameters, and the object that decides whether one uploaded dump matches the server's own configuration.

**Needs** — [`mp_config_sections.h`](mp_config_sections.h.md) · [`configs_common.h`](configs_common.h.md) · [`xrCore/Crypto/xr_dsa_verifyer.h`](../../xrCore/Crypto/xr_dsa_verifyer.h.md) · [`xrCore/xr_ini.h`](../../xrCore/xr_ini.h.md)

**Used by** — [`configs_dump_verifyer.cpp`](configs_dump_verifyer.cpp.md) · [`entry_point.cpp`](entry_point.cpp.md)

**Tier floor** — T2: a verdict over bytes. The signature primitive underneath it is T1, but nothing on this page is.

## Purpose

Declares the surface implemented in
[`configs_dump_verifyer.cpp`](configs_dump_verifyer.cpp.md). Two types:

`dump_verifyer` is the generic signature checker with this protocol's frozen parameters
already bound — it exists only so that the parameters are supplied in exactly one place
and cannot drift between call sites.

`configs_verifyer` is the verdict. It holds one expensive thing — the server's own
canonical dump, built once at construction and reused for every file checked — and one
cheap thing, the scratch space to append a per-dump suffix to it.

## Exported units

- `dump_verifyer` — the signature checker, bound to the parameters in
  [`configs_common.cpp`](configs_common.cpp.md).
- `configs_verifyer` — one verifier, reusable across many dumps.
- `verify` — the verdict: true when the dump matches, false with a human-readable reason
  when it does not.
- `verify_dsign` — recover the digest the client signed, or fail.
- `get_diff` / `get_section_diff` — name the first line that disagrees, for the failure
  message.
