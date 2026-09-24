# src/utils/mp_configs_verifyer/configs_common.cpp

> Holds the frozen public parameters of the configuration-dump signature scheme.

**Needs** — [`configs_common.h`](configs_common.h.md) · [`pch.h`](pch.h.md) · [`xrCore/Crypto/xr_dsa.h`](../../xrCore/Crypto/xr_dsa.h.md)

**Used by** — nothing in this recipe; entry point or dead code.

**Tier floor** — T1: fixed-width byte arrays handed to a cryptographic primitive that interprets them as big-endian integers.

## Purpose

A signature verifies only against the exact parameters and key it was produced under, so
those four numbers are frozen for as long as any shipped game client exists. They are
data, not code: this file is a table, and its entire content is four constants.

## State

```text
RECORD SignatureParameters          # all values fixed for the life of the protocol
  p   : bytes (128)    # the prime modulus; key width is 1024 bits
  q   : bytes (20)     # the prime divisor; 160 bits, matching the digest width
  g   : bytes (128)    # the generator
  key : bytes (128)    # the public key
```

**Invariants**

- The widths are not free. The narrow parameter is exactly the width of the digest the
  scheme signs, which is why the digest algorithm and the signature algorithm cannot be
  changed independently — see
  [`configs_dump_verifyer.cpp`](configs_dump_verifyer.cpp.md).
- All four are big-endian integers: the first byte is the most significant. Loading them
  in the machine's own byte order produces a key that verifies nothing and reports no
  error, only failures.
- They are shared with the game client, which carries the **private** half. That is the
  scheme's structural weakness and it is worth stating plainly: a signing key shipped
  inside the program it authenticates can be recovered by anyone who wants it, so the
  signature proves the dump came from *a* copy of the client, not from an unmodified one.
  The scheme raises the cost of cheating; it does not make it impossible. A rebuild that
  wants a real guarantee has to sign on a server the player does not control.

**Notes**

- Nothing here can be re-derived. If these numbers are lost, every dump ever signed
  becomes unverifiable, and the only recovery is to reissue them together with a new
  client. That is the sense in which they are frozen.
