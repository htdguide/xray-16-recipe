# src/xrCore/Crypto/xr_dsa.h

> Declares the signature primitive: a 1024-bit domain fixed at construction, a 20-byte private key, a 128-byte public key, sign and verify.

**Needs** — [`xr_dsa.cpp`](xr_dsa.cpp.md) · [`../xrstring.h`](../xrstring.h.md)
**Used by** — [`configs_common.cpp`](../../utils/mp_configs_verifyer/configs_common.cpp.md) · [`configs_common.h`](../../utils/mp_configs_verifyer/configs_common.h.md) · [`xr_dsa.cpp`](xr_dsa.cpp.md) · [`xr_dsa_signer.cpp`](xr_dsa_signer.cpp.md) · [`xr_dsa_signer.h`](xr_dsa_signer.h.md) · [`xr_dsa_verifyer.cpp`](xr_dsa_verifyer.cpp.md) · [`xr_dsa_verifyer.h`](xr_dsa_verifyer.h.md) · [`configs_common.cpp`](../../xrGame/configs_common.cpp.md) · [`configs_common.h`](../../xrGame/configs_common.h.md)
**Tier floor** — T1: the three domain numbers and both keys are fixed-width big-endian byte arrays whose lengths are compile-time constants other files assert against.

## Purpose

Declares the surface implemented in [`xr_dsa.cpp`](xr_dsa.cpp.md). The sizes it fixes are the interesting part, because they are asserted against from two other files and constrain the digest algorithm.

## Exported units

- **`key_bit_length`** — 1024. The size of the signature domain.
- **`public_key_length`** — 128 bytes, the bit length over eight. The domain's modulus, its generator, and any public key are all this wide.
- **`private_key_length`** — 20 bytes. The domain's subgroup order and any private key are this wide.
- **`private_key_t` / `public_key_t`** — fixed-size byte arrays, big-endian, most significant byte first.
- **`xr_dsa(p, q, g)`** — construct over a domain: modulus, subgroup order, generator.
- **`sign(private_key, data, size) -> text`** — a signature as uppercase hexadecimal.
- **`verify(public_key, data, size, signature) -> bool`**.

**Invariants** — `private_key_length` is 20 **because the digest is 20 bytes**, and the signing layer asserts the two are equal. The relationship runs the other way from how it looks: the subgroup order's width fixes the digest algorithm, not the reverse. Changing either without the other breaks the build, which is the right outcome.

The three domain numbers are supplied by the caller, not compiled in here — the game module carries them, together with the public key, and (on the signing side) fills the private key in a separate step. See [`xr_dsa_signer.h`](xr_dsa_signer.h.md).

**Notes** — the two key records are distinguished only by their length; nothing type-checks that a private key is not passed where a public one belongs, because both are byte arrays. A rebuild gets this for free with distinct types.
