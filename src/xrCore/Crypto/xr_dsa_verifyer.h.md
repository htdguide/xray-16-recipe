# src/xrCore/Crypto/xr_dsa_verifyer.h

> Declares the verify-then-report wrapper: a domain plus a public key in, a digest or nothing out.

**Needs** — [`xr_dsa_verifyer.cpp`](xr_dsa_verifyer.cpp.md) · [`xr_dsa.h`](xr_dsa.h.md) · [`xr_sha.h`](xr_sha.h.md)
**Used by** — [`configs_dump_verifyer.cpp`](../../utils/mp_configs_verifyer/configs_dump_verifyer.cpp.md) · [`configs_dump_verifyer.h`](../../utils/mp_configs_verifyer/configs_dump_verifyer.h.md) · [`xr_dsa_verifyer.cpp`](xr_dsa_verifyer.cpp.md) · [`configs_dump_verifyer.cpp`](../../xrGame/configs_dump_verifyer.cpp.md) · [`configs_dump_verifyer.h`](../../xrGame/configs_dump_verifyer.h.md) · [`gsc_dsigned_ltx.h`](../../xrGame/gsc_dsigned_ltx.h.md)
**Tier floor** — T2: a digest call and a verify call.

## Purpose

Declares the surface implemented in [`xr_dsa_verifyer.cpp`](xr_dsa_verifyer.cpp.md). Unlike its signing counterpart, the public key **is** a constructor argument — there is nothing to hide about it — so this type is usable directly and is subclassed only to bind a particular domain.

## Exported units

- **`xr_dsa_verifyer(p, q, g, public_key)`** — construct over a signature domain and the key to check against. Copies the key in.
- **`m_public_key`** — the 128-byte key, protected so a subclass that supplies its own can see it.
- **`verify(data, size, signature) -> optional<digest>`** — check the signature over the buffer; on success yield the digest that was verified, on failure yield nothing.

**Notes** — returning the digest rather than a boolean is the file's one real design decision, and it is a good one: the caller almost always wants the digest afterwards — to record what it accepted, or to compare against a digest transmitted separately — and this way it cannot obtain one without having verified it. The type makes "I have a digest" and "the signature checked out" the same fact.
