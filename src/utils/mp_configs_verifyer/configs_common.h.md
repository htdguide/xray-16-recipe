# src/utils/mp_configs_verifyer/configs_common.h

> Declares the four fixed numbers that constitute the signature scheme's public half.

**Needs** — [`xrCore/Crypto/xr_dsa.h`](../../xrCore/Crypto/xr_dsa.h.md)

**Used by** — [`configs_common.cpp`](configs_common.cpp.md) · [`configs_dump_verifyer.cpp`](configs_dump_verifyer.cpp.md) · [`configs_dump_verifyer.h`](configs_dump_verifyer.h.md)

**Tier floor** — T1: they are byte arrays of an externally fixed width, fed to a cryptographic primitive.

## Purpose

Declares the surface defined in [`configs_common.cpp`](configs_common.cpp.md): the three
domain parameters of the signature scheme and the public key that goes with them. They are
in a header of their own so that the verifier and anything else built from this tree bind
to the same numbers — a signature is only meaningful against the exact parameters it was
made under.

## Exported units

- `p_number`, `g_number` — the two wide domain parameters, each the full key width.
- `q_number` — the narrow domain parameter, the width of the digest the scheme signs.
- `public_key` — the public half of the pair whose private half lives in the game client.
