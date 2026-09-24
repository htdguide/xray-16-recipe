# src/xrGame/configs_common.cpp

> The signature parameters shared by the client that signs a configuration dump and the server that checks it.

**Needs** — [`configs_common.h`](configs_common.h.md) · [`xrCore/Crypto/xr_dsa.h`](../xrCore/Crypto/xr_dsa.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: fixed-width byte arrays whose exact contents are the contract

## Purpose

The multiplayer anti-cheat scheme has a client produce a signed image of the configuration
it is running with and a server verify it — see
[`configs_dumper.cpp`](configs_dumper.cpp.md) and
[`configs_dump_verifyer.cpp`](configs_dump_verifyer.cpp.md). Both ends need the same
signature domain and the same verification key, so those constants live here, in the one
file both include.

## State

```text
p : bytes (128)   # the signature scheme's prime modulus, big-endian
q : bytes (20)    # its prime divisor
g : bytes (128)   # the generator
public_key : bytes (128)   # the verification key
```

**Invariants** — the four arrays are a matched set: they were generated together and no
subset of them is meaningful. Their lengths are fixed by the scheme — a 1024-bit modulus and
a 160-bit divisor — and the digest the signature is computed over is 160 bits to match.

**Notes** — these are literal byte tables with no derivation in the repository. They cannot
be regenerated, because the *private* key that matches this public key is also baked into the
source (see [`configs_dumper.cpp`](configs_dumper.cpp.md)) and a rebuild that generates a
fresh pair is incompatible with every existing client and server. A rebuild must copy these
bytes verbatim if it wants to interoperate, and should treat them as a wire format rather
than as a secret.

**Notes** — because the private key ships in the client, **the signature does not
authenticate anything**. Anyone with the binary can sign an arbitrary dump. What the scheme
actually buys is a barrier against tampering with the dump *in transit* and against the
naive attack of editing a configuration file and connecting; it is not a defence against
someone who modifies the executable. The real check is the content comparison the verifier
does, not the signature.
