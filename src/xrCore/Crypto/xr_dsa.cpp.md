# src/xrCore/Crypto/xr_dsa.cpp

> Signs and verifies a 20-byte digest against a 1024-bit signature domain, exchanging signatures as hexadecimal text. Reduces to "always fails" when the engine is built without a cryptography provider.

**Needs** — [`xr_dsa.h`](xr_dsa.h.md) · [`../xrstring.h`](../xrstring.h.md) · [`../log.h`](../log.h.md)
**Used by** — [`xr_dsa.h`](xr_dsa.h.md)
**Tier floor** — T1: fixed-width big-endian key material handed across a foreign boundary, and a signature buffer sized by asking the provider and then allocated on the stack.

## Purpose

The only thing the engine signs is its multiplayer configuration dump, so that a client can confirm the server is running unmodified rules. This file is the signature primitive; [`xr_dsa_signer.cpp`](xr_dsa_signer.cpp.md) and [`xr_dsa_verifyer.cpp`](xr_dsa_verifyer.cpp.md) are the digest-then-sign and verify-then-report wrappers around it.

The algorithm itself is not implemented here. It is an **external cryptography provider**, and the entire file is conditional on one being present.

## State

```text
RECORD Signer
  key     : provider key object     # holds the domain, and whichever of the two
                                    #   key halves was most recently installed
  context : provider operation context
```

**Invariants** — there is one key object and one context per instance, and **both sign and verify mutate them**: each installs its key half into the same key object before operating. So one instance cannot be used concurrently from two threads, and a verify following a sign on the same instance leaves the private half still resident. Nothing in the engine does either, because the signer and the verifier are separate objects that each only ever do one of the two.

## Construction

**Contract** — take the three domain numbers as big-endian byte arrays of the declared widths, convert them into the provider's number representation, and build a key object holding only the domain parameters — no key half yet. Releases the temporaries. Does not validate the domain and does not report failure: a malformed domain surfaces later as a signature that never verifies.

```text
FUNCTION construct(p, q, g)
  context := new operation context for the signature algorithm
  key := build from parameters {
           modulus:        big_endian_number(p, 128 bytes),
           subgroup_order: big_endian_number(q,  20 bytes),
           generator:      big_endian_number(g, 128 bytes)
         }
```

**Invariants** — every key and domain number crosses this boundary **most significant byte first**, regardless of the machine's own byte order. That is the one endianness exception in a codebase that is otherwise little-endian throughout ([`SYSTEM-REQUIREMENTS.md` §4](../../../SYSTEM-REQUIREMENTS.md#4-platform-assumptions)), and forgetting it produces a domain that is silently wrong.

## `sign`

**Contract** — install the private key half into the existing key object, ask the provider how large a signature will be, sign the caller's bytes into a scratch buffer of that size, and return the signature as **uppercase hexadecimal text** with no leading zeros. The caller's bytes are the digest, not the message. Allocates the scratch buffer on the stack, so a provider reporting an implausible size is a stack overflow rather than an error. Without a provider, logs a warning and returns the empty string.

```text
FUNCTION sign(private_key, digest, n) -> text
  install private_key into the key object
  begin a signing operation
  size := ask the provider for the signature length
  buffer := scratch of `size` bytes
  sign digest into buffer
  RETURN hexadecimal_of(big_endian_number(buffer))
```

**Notes** — round-tripping the signature through a big number and back to hexadecimal is how the engine gets a printable form it can write into a configuration file. It is also lossy in a way that matters: **a signature with leading zero bytes loses them**, because a number has no leading zeros. Verification re-parses the hexadecimal into a number and converts back to bytes, so it loses the same bytes and the two sides agree — but the signature's *byte length* is then not fixed, and any code that assumes it is will be wrong roughly once in 256 signatures. A rebuild should carry the signature as fixed-width bytes and convert to text only for display.

The source also contains a real defect on this path: the scratch number used for the conversion is created by the sign step and reused without being reinitialized in the failure ordering. A rebuild writing this fresh will not reproduce it.

## `verify`

**Contract** — parse the hexadecimal signature back into bytes, install the public key half, and ask the provider whether the signature covers the caller's digest. Returns true only on an explicit affirmative from the provider. Without a provider, logs a warning and returns false.

```text
FUNCTION verify(public_key, digest, n, signature_text) -> bool
  bytes := big_endian_bytes(number_from_hexadecimal(signature_text))
  install public_key into the key object
  begin a verification operation
  RETURN the provider says the signature is valid
```

**Invariants** — the no-provider path **fails closed** here, which is the correct direction and the opposite of what the digest layer does ([`xr_sha.cpp`](xr_sha.cpp.md) returns an all-zero digest that compares equal to itself). The combination is the hazard: a build with no provider produces matching all-zero digests but a verify that always returns false, so it fails closed overall — by accident rather than by design.

**Notes** — the whole scheme's security rests on a private key that ships inside a client binary and is assembled at run time by a function whose only purpose is to make it less greppable. That is obfuscation, and the recipe should not present it as more. What a rebuild must preserve is the *format*: the domain widths, the big-endian key material, and the hexadecimal signature text, because signed configuration files written by the original engine must still verify.
