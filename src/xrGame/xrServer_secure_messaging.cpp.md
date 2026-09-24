# src/xrGame/xrServer_secure_messaging.cpp

> Establishes a per-client shared secret and wraps chosen messages in it, so that a message a client must not forge cannot be forged.

**Needs** — [`xrServer.h`](xrServer.h.md) · [`xrMessages.h`](../xrServerEntities/xrMessages.h.md) · [`secure_messaging.h`](secure_messaging.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: encrypts a byte range of a packet in place at a fixed offset

## Purpose

Most traffic in this protocol is unauthenticated and the server compensates with validation. Some
of it cannot be validated — a client reporting something only it can know — and for those the
protocol needs a channel a third party cannot write to and a client cannot replay. This file is
that channel: a seed exchange that derives a per-client key, and an envelope that carries an
ordinary message encrypted under it.

**Treat the scheme as documented, not as endorsed.** The weaknesses below are load-bearing for a
rebuild precisely because reproducing them would be a mistake.

## State

`Stateless.` — the per-client key and the outstanding seed live on the client record; see
[`xrServer.h`](xrServer.h.md).

## `PerformSecretKeysSync`

**Contract** — begin a key exchange with one client. Generates a fresh 32-bit seed, records it on
the client's record as outstanding, and sends it to the client. **Does not derive the key yet** —
derivation waits for the acknowledgement.

**Notes** — **the seed is sent in the clear.** Anyone who can observe the connection can derive the
same key. The scheme therefore protects against a *client* forging messages, not against an
observer — it is an anti-cheat measure, not a confidentiality measure, and calling it secure
messaging oversells it. A rebuild wanting either property should use a real key agreement; wanting
only the anti-cheat property, a server-chosen secret sent over an already-authenticated channel is
enough and this is very nearly that.

The seed is 32 bits, which bounds the key space regardless of how the key is derived from it.

## `PerformSecretKeysSyncAck`

**Contract** — the client's acknowledgement. Reads back the seed, **checks it matches the
outstanding one only in checked builds**, and derives the key from the *server's* recorded seed —
never from the value the client sent.

**Invariants** — deriving from the recorded seed rather than the received one is the line that
makes the echo harmless. A client that echoes a different seed gets a key that does not match the
server's and every subsequent envelope it sends fails its checksum. The echo is a round-trip
confirmation, not an input.

**Notes** — the mismatch check exists only in checked builds and, when it fires, is a hard failure
rather than a disconnect. In a shipping build a mismatch is silently ignored and the client simply
stops working. A rebuild should disconnect the client and log once.

## `SecureSendTo`

**Contract** — send one message to one client inside an envelope. Wraps the message's bytes in an
envelope message, encrypts everything after the envelope's own type field under that client's key,
appends the encryption's checksum, and sends.

```text
FUNCTION secure_send_to(client, message, flags)
  envelope := begin(SECURE_MESSAGE) + message.bytes
  checksum := encrypt_in_place(envelope.bytes after the 2-byte type, client.key)
  envelope += checksum (32-bit)
  send_to(client, envelope, flags)
```

**Invariants** — the envelope's own type field stays in the clear, because the receiver must read
it to know the message is an envelope at all. Everything after it is ciphertext, including the
inner message's type.

## `OnSecureMessage`

**Contract** — unwrap a received envelope and feed the inner message back through the ordinary
message switch as though it had arrived plain. Decrypts, compares the computed checksum against the
transmitted one, and **drops the message silently on a mismatch**.

```text
FUNCTION on_secure_message(packet, sender)
  inner_length := packet.length - 2 (the type) - 4 (the trailing checksum)
  inner := packet.read_bytes(inner_length)
  computed := decrypt_in_place(inner, sender.key)
  transmitted := packet.read_int(32-bit)
  IF computed != transmitted THEN RETURN      # no log; see Notes
  on_message(inner, sender)
```

**Invariants** — **the failure path logs nothing, deliberately**, and the source says why: a log
line is an oracle. A client probing keys would learn from the server's own output which attempts
got further, so the server says nothing at all and the attacker learns only that nothing happened.
That is the right instinct and a rebuild should keep it — with the caveat that an *aggregate*
counter, not per-message, gives an operator the visibility they need without the oracle.

The inner length is computed by subtracting two known fixed sizes from the packet's length, so
both the type width and the checksum width are frozen in the format.

**Notes** — the checked build performs a self-test on every received envelope: it encrypts and then
decrypts a fixed string under the client's key and asserts the two checksums agree. That is a test
of the primitive, not of the message, and running it per message is a debug-build cost only. What
it tells a reader is that the primitive is symmetric in a way the author did not fully trust — a
rebuild should test the primitive once, in a test.

Feeding the decrypted message back through the ordinary switch means **any message type can travel
inside an envelope**, and nothing marks which types *must*. So the envelope is available but not
required, and a client can send a sensitive message in the clear. A rebuild should have the switch
refuse the message types that require authentication when they arrive unwrapped.
