# src/utils/mp_configs_verifyer — the configuration anti-cheat

Part of chapter 28 of [the build order](../../../SYSTEM-REQUIREMENTS.md#7-build-order).

## What this module is responsible for

Deciding whether a multiplayer client was running the server's balance data. A client
uploads a signed, compressed dump of the configuration sections that decide fairness; this
tool rebuilds the same dump from the server's own data, compares digests through the
client's signature, and — when they differ — names the first line that disagrees.

It is the **server half of a two-sided protocol**. The client half lives in the game
([chapter 23](../../xrGame/README.md)); the shared vocabulary, the dump's layout and the
signed byte region are described here, because this is where the specification of the
format actually is.

## Where it sits

At the end, beside the packer and the balance tool. It rests on the core layer only — the
virtual filesystem, the configuration parser, the digest and signature primitives, and the
high-ratio compressor — and nothing rests on it. It runs beside a dedicated server, as a
long-lived filter process that a server driver pipes file names into.

## Load-bearing ideas, named once

**The protected set is drawn two ways at once.** Most of it is read out of the shipped
configuration itself, from the section that lists the item groups — so adding an item to
the game adds it to the anti-cheat set with no code change. The rest is a frozen list of
twenty-three names covering the things that are not items: the actor's parameters, the
rank ladder, the per-mode and per-team rules, and the two bonus tables. That list is the
shortest honest statement of what the designers considered exploitable.

**Order is the contract.** The protected set is serialized in a fixed order — item-group
members in file order, then the fixed names in source order — and the digest is taken over
the concatenation. Two ends that build the same *set* in a different *order* report every
honest player as a cheat.

**What is hashed is the rendered text, not the values.** The configuration writer's column
padding, its separator spacing and its blank line between sections are all part of the
protocol. Change the writer and every existing dump stops verifying. This coupling is
entirely implicit in the original and is the most fragile thing in the directory.

**The dump has a static half and a dynamic half.** The static half is the protected set.
The dynamic half describes what the player is actually carrying, and the dump supplies
only the *names* of those sections — the verifier looks up what they should say from its
own configuration. A client that alters a value is caught; a client that lies about a name
is caught because the name list is itself inside the digest.

**The identity triple binds a dump to a player and a moment**: player name, player digest
and creation date, concatenated with no separator, zero-terminated, appended after
everything else. Without it one valid dump could be replayed forever.

**The signature cannot cover itself, so the signed region is everything before the
identity section plus the triple.** The verifier reconstructs that region by truncating
the uploaded buffer at the identity section's opening bracket and appending the triple
there. One byte either way changes the digest.

**The signing key ships inside the client.** The verifier holds only the public half, but
the private half is in the program being authenticated, so the signature proves a dump
came from *a* copy of the client, not from an unmodified one. The scheme raises the cost
of cheating; it does not make it impossible. A rebuild wanting a real guarantee signs on
a machine the player does not control.

**The parse is of hostile input and is wrapped in a fault barrier.** A crash inside the
verdict is caught, reported as a failure, and the filter continues. A rebuild that
bounds-checks its parse does not need the barrier, but must keep the property it buys:
**one bad dump must not stop the stream.**

**The authoritative configuration is loaded once and never reloaded.** A server that edits
its balance while the verifier is running reports every subsequent honest dump as a cheat
until the verifier restarts. That operational constraint is nowhere stated in the original.

## The files

| File | Role |
|---|---|
| [`configs_dump_verifyer.cpp`](configs_dump_verifyer.cpp.md) | The verdict: the dump's layout, the signed region, the digest comparison, the difference report |
| [`configs_dump_verifyer.h`](configs_dump_verifyer.h.md) | Its surface |
| [`mp_config_sections.cpp`](mp_config_sections.cpp.md) | The protected set: how it is enumerated, its fixed members, and the serialization order |
| [`mp_config_sections.h`](mp_config_sections.h.md) | Its surface, and the dynamic half's two sides |
| [`configs_common.cpp`](configs_common.cpp.md) | The frozen public signature parameters |
| [`configs_common.h`](configs_common.h.md) | Their declaration |
| [`entry_point.cpp`](entry_point.cpp.md) | The three modes: one file, a name stream, unpack-and-stop; and the compressed dump's own four-byte framing |
| [`pch.h`](pch.h.md) · [`pch.cpp`](pch.cpp.md) | Build-time header aggregation; no decisions |

Three further files in the directory are build description and carry no decisions.
