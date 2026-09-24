# src/xrGame/configs_common.h

> Declares the shared signature parameters defined in [`configs_common.cpp`](configs_common.cpp.md).

**Needs** — [`xrCore/Crypto/xr_dsa.h`](../xrCore/Crypto/xr_dsa.h.md)
**Used by** — [`configs_common.cpp`](configs_common.cpp.md) · [`configs_dump_verifyer.cpp`](configs_dump_verifyer.cpp.md) · [`configs_dumper.cpp`](configs_dumper.cpp.md)
**Tier floor** — T3: four declarations

## Purpose

Declares the four constant byte arrays defined in
[`configs_common.cpp`](configs_common.cpp.md), so that the signer and the verifier see the
same domain.

Exported units:

- **`p_number`, `q_number`, `g_number`** — the signature scheme's domain parameters.
- **`public_key`** — the verification key.
