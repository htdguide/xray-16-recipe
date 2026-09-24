# src/xrCore/Crypto — one obfuscator and one signature chain

Part of chapter 6, [`src/xrCore`](../README.md). Ten files serving two entirely unrelated
purposes, and it is worth saying at the top that neither of them is security in any sense a
modern reader would accept.

## What this module is responsible for

**Reading obfuscated archive directories.** The later game generations store their archive
directory chunk behind a byte permutation and a keystream. Two keys shipped — one for the
Russian release, one for the international — and the engine does not know which archive it
has, so it tries one, attempts to decompress, and retries with the other. This is not
optional: it is how the shipped data is encoded, and the engine exists to read the shipped
data.

**Signing a multiplayer configuration dump.** A server publishes a digest-and-signature over
its configuration so a client can confirm the rules match. The signature domain and the
public key ship in the game module; the private key ships in the same binary, assembled at
run time to make it marginally less greppable. As anti-tamper this is defeated by anyone who
reads the binary; as a consistency check between an unmodified client and an unmodified
server it works.

The module owns the byte-level formats of both — the permutation's key constants, the
signature's key widths and its hexadecimal text form, and the timestamp layout in the signed
dump — and nothing else.

## Where it sits

The obfuscator rests only on the engine's small random generator
([`../Math/Random32.hpp`](../Math/Random32.hpp.md)) and is consumed by the virtual
filesystem ([`../LocatorAPI.cpp`](../LocatorAPI.cpp.md)) during mounting. The signature
chain rests on an **external cryptography provider** that is not a declared seam in
[`SYSTEM-REQUIREMENTS.md`](../../../SYSTEM-REQUIREMENTS.md) because it is optional: the
whole subsystem compiles to stubs without it. It is consumed by chapter 23's multiplayer
anti-tamper path and by the standalone configuration-verifier tool in chapter 28.

## The load-bearing ideas

**The obfuscator's constants are the format.** Two iteration counts, four seeds, and the
exact arithmetic of the generator that drives them. Substituting a different generator, or
reducing its range by a remainder instead of a widening multiply, decodes every shipped
archive to garbage. [`trivial_encryptor.cpp`](trivial_encryptor.cpp.md) prints them.

**There is no marker saying which key an archive used.** Successful decompression is the
only oracle. That is why the filesystem's mount path decodes, tries, re-encodes and retries.

**The keystream is positional, not streaming.** It restarts from the key's seed at every
call, so byte *i* of every buffer gets the same keystream byte. Decoding a buffer in two
calls gives the wrong answer for the second half.

**The digest width is not a choice.** The signature domain's subgroup order is 160 bits, so
the private key is 20 bytes, so the digest must be 20 bytes. The relationship is asserted at
build time in [`xr_dsa_signer.cpp`](xr_dsa_signer.cpp.md) and runs from the domain outward.

**Key material crosses the provider boundary big-endian**, in a codebase that is otherwise
little-endian everywhere ([`SYSTEM-REQUIREMENTS.md` §4](../../../SYSTEM-REQUIREMENTS.md#4-platform-assumptions)).
This is the exception, and it is silent when wrong.

**Without a provider, the digest layer fails *open* and the signature layer fails *closed*.**
An absent provider returns an all-zero digest — which compares equal to itself — and a verify
that always returns false. The combination happens to be safe; neither half is safe alone. A
rebuild should make the absent provider a build failure and make "no digest" a value that no
comparison accepts.

## The twins

| File | Role |
|---|---|
| [`trivial_encryptor.h`](trivial_encryptor.h.md) | Declares the archive obfuscator, its two keys, and the one shared instance. |
| [`trivial_encryptor.cpp`](trivial_encryptor.cpp.md) | **The obfuscation format**: the permutation build, the keystream, and the six constants. Frozen. Substantive. |
| [`xr_sha.h`](xr_sha.h.md) | Declares the 20-byte digest and the per-block yield hook. |
| [`xr_sha.cpp`](xr_sha.cpp.md) | The block loop, the yield point, and the fail-open policy a rebuild must not copy. |
| [`xr_dsa.h`](xr_dsa.h.md) | Declares the signature primitive and the three widths everything else asserts against. |
| [`xr_dsa.cpp`](xr_dsa.cpp.md) | **Sign and verify**: big-endian key material, hexadecimal signature text, and the leading-zero loss that text costs. Substantive. |
| [`xr_dsa_signer.h`](xr_dsa_signer.h.md) | Declares the digest-then-sign wrapper and the private-key slot a subclass fills. |
| [`xr_dsa_signer.cpp`](xr_dsa_signer.cpp.md) | Digest, sign, and the timestamp format the signed dump carries. |
| [`xr_dsa_verifyer.h`](xr_dsa_verifyer.h.md) | Declares the verifier, which returns the digest it verified rather than a boolean. |
| [`xr_dsa_verifyer.cpp`](xr_dsa_verifyer.cpp.md) | Digest, verify, report — and the diagnostic dump that is the only way to localize a mismatch. |
