# src/xrGame/Level_secure_messaging.cpp

> Wraps a message in an obfuscating cipher with a checksum, unwraps received ones, and re-derives the shared key whenever the server sends a new seed.

**Needs** — [`Level.h`](Level.h.md) · [`NET_Queue.h`](NET_Queue.h.md) · [`xrNetServer/NET_Messages.h`](../xrNetServer/NET_Messages.h.md)
**Used by** — [`Level_network_digest_computer.cpp`](Level_network_digest_computer.cpp.md)
**Tier floor** — T1: the cipher operates on the packet's raw byte buffer in place, and the size arithmetic is in terms of the wire header's exact widths

## Purpose

A handful of messages — the client's identity digest, scoring and administrative traffic —
are sent through a cipher rather than in the clear. The purpose is not confidentiality
against an observer; it is to make casual packet forgery by a cheating client inconvenient.
The server periodically reseeds the key and the client acknowledges, so a key recovered
once does not stay valid.

Three operations is the whole file: wrap, unwrap, reseed.

## State

```text
RECORD SecureChannel               # lives on the level
  secret_key : opaque              # derived from the server's seed; symmetric, shared
```

**Invariants** — both ends must derive the same key from the same seed, and must not use the
new key until the reseed has been acknowledged, or messages in flight decrypt to garbage.

## `SecureSend`

**Contract** — takes a fully built message, encrypts its bytes in place under the current
key, and sends the ciphertext plus its checksum as the payload of a secure-message
envelope. Passes the caller's reliability flags and timeout straight through to the
transport. Allocates one packet-sized buffer on the stack.

```text
FUNCTION SecureSend(message, flags, timeout)
  envelope = new packet beginning with the SECURE_MESSAGE identifier
  checksum = encrypt(message.bytes, message.length, secret_key)   # in place
  append message.bytes to envelope
  append checksum        : int (32-bit)
  send envelope with flags and timeout
```

## `OnSecureMessage`

**Contract** — the receive half. Peels the envelope, decrypts the payload, compares the
computed checksum against the transmitted one, and hands the recovered message to the
level's pending game-event queue rather than dispatching it immediately.

```text
FUNCTION OnSecureMessage(envelope)
  payload_length = envelope.length - sizeof(message identifier) - sizeof(checksum)
  read payload_length bytes
  computed = decrypt(payload, payload_length, secret_key)
  read transmitted checksum
  REQUIRE computed == transmitted          # a mismatch means forged or corrupted input
  enqueue the recovered message on the game-event queue
```

**Invariants** — the payload length is derived by subtracting the two fixed-width fields from
the envelope's length. That arithmetic is the format: an envelope is *identifier, ciphertext,
checksum*, with the ciphertext's length implied.

**Notes** — the checksum comparison is the security boundary and it is only checked in
non-shipping builds; the shipping build enqueues whatever it decoded. A mismatched message
then fails somewhere later and unrecognizably. A rebuild must make this a hard, always-on
rejection that drops the packet and logs, not an assertion.

The recovered message goes on the queue rather than being dispatched inline, so a secure
message is handled at the same point in the frame as an ordinary one. Without that, secure
and ordinary messages sent in one batch would be applied out of order.

## `OnSecureKeySync`

**Contract** — reads a new seed from the server, derives the shared key from it, and echoes
the seed back as an acknowledgement. Reliable and ordered.

**Notes** — the seed is echoed rather than a derived value, which means the acknowledgement
proves only that the message arrived, not that the client derived the same key. The comment
in the original says the echoed field is for debugging, which is an admission that the
acknowledgement carries no verification. A rebuild that wants the check should echo a
digest of the derived key instead.

The seed is a 32-bit signed integer, which is the whole key space: this scheme is
obfuscation, not cryptography, and should be described as such wherever it is offered to a
user.
