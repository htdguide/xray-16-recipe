# src/xrGame/screenshot_server.cpp

> Demands a screenshot or a configuration dump from one client and forwards it to the administrator who asked, starting the forward leg before the download has finished.

**Needs** — [`screenshot_server.h`](screenshot_server.h.md) · [`file_transfer.h`](file_transfer.h.md) · [`xrMessages.h`](../xrServerEntities/xrMessages.h.md) · [`Level.h`](Level.h.md) · [`xrServer.h`](xrServer.h.md) · [`game_sv_base.h`](game_sv_base.h.md) · [`game_cl_mp.h`](game_cl_mp.h.md) · [`xrNetServer/NET_Messages.h`](../xrNetServer/NET_Messages.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: two chained streamed transfers driven by callbacks

## Purpose

The server half of the anti-cheat demand. Three decisions are made here and nothing else is:
what the request packet looks like on the wire, when the forward leg to the administrator
starts, and what the server keeps on disk.

Read the system requirements' networking seam first. The shipped build has no working
transport, so **this code is reachable only in a build that turns multiplayer on**; it is
documented as a design, not as a running feature.

## State

See [`screenshot_server.h`](screenshot_server.h.md).

## `make_screenshot` / `make_config_dump`

**Contract** — issue a demand. Both have the same body and differ only in the request byte
and in which receive callback is installed. Resolve the target client; refuse with a log line
if it is not connected, or if a receive from it is already running. Capture its name and key
digest. Send the request over the secure channel, reliably. Clear the buffer, mark the
forward leg unstarted, and open a receive into the buffer.

```text
FUNCTION demand(admin, target, kind)
  client = server.client(target)
  IF client DOES NOT EXIST THEN log("client not found"); RETURN
  IF transport.is_receiving_from(target) THEN log("already active, try later"); RETURN

  target_name   = client.player_name OR "unknown"
  target_digest = client.key_digest

  packet = game_message(MAKE_DATA)
  packet.write_byte(kind)              # SCREENSHOT_REQUEST or CONFIGS_REQUEST
  packet.write_int16(random 0..1) x3   # padding — see invariants
  packet.write_int8 (random 0..1)
  server.secure_send_to(client, packet, reliable)

  buffer.clear()
  first_receive = true
  receiver = transport.start_receive(buffer, FROM target, ON kind's callback)
```

**Invariants** — the four random values are **padding, and their purpose is to make this
packet indistinguishable by length from the player-killed message.** The comment in the
source says so outright. A demand that a cheat client could recognize by packet size would be
a demand it could suppress or fake, so the request is disguised as ordinary game traffic. The
values are random rather than zero so the packet does not compress to a recognizable shape
either. A rebuild that drops the padding loses the only protection this mechanism has, and a
rebuild that changes the killed-message layout must re-match it — the two sizes are coupled
and nothing enforces the coupling.

**Notes** — the disguise is weak by modern standards (the transport is not encrypted, only
the send path is called "secure"), and the whole feature rests on the client's own code
answering honestly, which a cheat client need not. Treat it as a deterrent against
unmodified-but-misbehaving clients, not as an anti-cheat guarantee.

## The receive callbacks

**Contract** — one per kind, structurally identical, called by the transfer service as the
target's file arrives. Four statuses are handled:

- **data arriving** — log progress and, **on the first chunk only, start the forward leg to
  the administrator**;
- **aborted locally** — terminate the process. See the notes;
- **aborted by the peer** — log and notify the administrator with a reason naming the peer;
- **timed out** — log and notify the administrator;
- **complete** — start the forward leg if it somehow has not started, and then write the file
  to disk if the corresponding server setting is on.

```text
FUNCTION on_receive(status, downloaded, total)
  IF status IS data_arriving OR status IS complete THEN
    IF first_receive THEN
      notify_admin(kind's RESPONSE, "prepare for receive...")
      transport.start_transfer(buffer, total, TO admin, FROM target, ON upload callback,
                               user_param = receiver.user_param)
      first_receive = false
    IF status IS complete AND server_setting_says_keep THEN write_to_disk()
  ELSE
    notify_admin(kind's ERROR, describe(status))
```

**Invariants**

- **The forward leg starts on the first arriving chunk, not on completion.** The transfer
  service is told the total size up front and streams out of the same buffer the download is
  streaming into, so the administrator's copy begins while the target is still uploading.
  This halves the administrator's wait and is the reason the buffer is shared rather than
  copied — and it is also the reason the buffer must not be reallocated in a way that
  invalidates the sender's view. A rebuild that forwards only on completion is simpler and
  slower; one that keeps the streaming must make the buffer's growth safe for a concurrent
  reader.
- The completion branch repeats the start, guarded by the same flag, to cover a file that
  arrives in a single chunk reported only as complete. The flag is what makes both entries
  idempotent.
- The **user parameter travels with the transfer and carries the uncompressed size** of the
  payload. Both the screenshot and the configuration are sent compressed by the client; the
  server neither decompresses nor inspects them on the relay path, so this one number is all
  it knows about the content. It is forwarded verbatim to the administrator and written into
  the configuration file's header.

**Notes** — the locally-aborted case terminates the process rather than cleaning up. That is
a deliberate "cannot happen" assertion — nothing on the server cancels a receive — but it
means a transport that reports cancellation for any other reason kills the server. A rebuild
should log and abandon the demand.

## `notify_admin`

**Contract** — sends one message to the administrator: the event kind, the target's client
identifier, and then either the target's **name** (on the two success kinds) or a **reason
string** (on everything else). Reliable. The two payload shapes share one message kind and
are told apart by the event byte, which the administrator's client must switch on before
reading the string.

## `is_active`

**Contract** — true while either leg is running: the download from the target or the upload
to the administrator. Used by the owner to decide whether this demand may be discarded.

## `save_proxy_screenshot`

**Contract** — writes the received screenshot into the screenshots directory, under a name
built from the target's player name and key digest, with the digest replaced by a placeholder
when the client has none. Decompression is delegated to the multiplayer game mode, which
knows the screenshot's encoding; the uncompressed size comes from the transfer's user
parameter. Does nothing when the server is not running a multiplayer game mode.

## `save_proxy_config`

**Contract** — writes the received configuration into the screenshots directory under the
target's name with a configuration extension, **still compressed**, prefixed by the
uncompressed size as one word. Unlike the screenshot it is not decoded, so the stored file is
a private format: size, then compressed bytes. A tool reading these back must know that.

**Notes** — both saves are gated by separate server settings, off by default. They exist so
an administrator can keep evidence; the relay to the administrator happens regardless. The
shared destination directory means configuration dumps land among the player's own
screenshots, which is surprising but harmless.

## Destruction

**Contract** — if either leg is still running, stop it: the receive from the target and the
transfer to the administrator, each checked before being stopped. The buffer is released with
the record. This is the only cleanup path, so a demand must be destroyed rather than
abandoned.
