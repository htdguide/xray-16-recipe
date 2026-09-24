# src/xrGame/xrServer_info.cpp

> Sends a joining client the server's logo and rules text over the file-transfer channel, and uses the transfer's completion as the signal that configuration is finished.

**Needs** — [`xrServer_info.h`](xrServer_info.h.md) · [`xrServer.h`](xrServer.h.md) · [`Level.h`](Level.h.md) · [`file_transfer.h`](file_transfer.h.md) · [`xrNetServer/NET_Messages.h`](../xrNetServer/NET_Messages.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — [`xrServer_info.h`](xrServer_info.h.md)
**Tier floor** — T1: hands raw byte ranges of open files to a transfer layer

## Purpose

A server operator can put an image and a page of rules beside the executable and every joining
player sees them. Mechanically this is the only place in the join sequence that transfers *bulk*
data, and it is therefore also the step that takes measurable time — which is why the join's
"configuration finished" signal is tied to its completion rather than sent independently.

## State

The two files, held open by the server: see [`xrServer.h`](xrServer.h.md). The uploader's own
state is in [`xrServer_info.h`](xrServer_info.h.md).

```text
logo_filename  = "server_logo.jpg"     # beside the executable, in the application data root
rules_filename = "server_rules.txt"
```

**Notes** — the names are fixed rather than configured, which is the right call for something a
server operator drops in a directory. The logo's format is not verified — the source flags it —
so a client receives whatever bytes are in the file and decodes them at its own risk.

## `LoadServerInfo`

**Contract** — open both files, once, at server start. **All or nothing**: if either is missing,
neither is loaded and the feature is simply off. If the logo opens and the rules do not, the logo
is closed again.

**Notes** — requiring both is a limitation the source itself flags: rules without a logo is a
reasonable configuration and is refused. A rebuild should make each independent.

## `SendServerInfoToClient`

**Contract** — begin the info transfer to one joining client. **Three paths and all three end at
the same place**: in single player, or with no info loaded, the configuration-finished signal is
sent immediately; otherwise a message telling the client to expect a logo is sent, an uploader is
obtained, and the transfer is started with the configuration-finished signal as its completion
callback.

```text
FUNCTION send_server_info_to(client)
  IF the game is single-player          THEN send_config_finished(client); RETURN
  IF logo or rules were not loaded      THEN send_config_finished(client); RETURN

  tell the client to expect a logo, naming the host's client as the sender
  uploader := a free uploader from the pool, or a new one
  uploader.start(logo, rules, client, on_complete: send_config_finished)
```

**Invariants** — **the configuration-finished signal is sent exactly once per client on every
path**, including when the transfer is rejected or aborted. A client that never receives it stays
on its loading screen forever, so every terminal state of the uploader must fire the callback —
and every one does.

**Notes** — the source marks this thread-unsafe, which matters because it is reached from the
client-connected path. The pool scan and the possible append are the unguarded part. A rebuild
running the join sequence off the simulation thread must guard it.

The "expect a logo" message names the *sending* client, which the receiver needs in order to match
the incoming transfer to this announcement — transfers are addressed by a (recipient, sender)
pair.

## `GetServerInfoUploader`

**Contract** — hand out an idle uploader, creating one if every existing one is busy. The pool
therefore grows to the high-water mark of concurrent joins and never shrinks.

**Notes** — an unbounded pool of one-shot workers is the simplest thing that cannot deadlock, and
the bound in practice is the maximum player count. A rebuild may prefer a fixed pool with a queue;
the failure mode to avoid is refusing a join because no uploader was free.

## `start_upload_info`

**Contract** — begin one transfer. Gathers the two payloads into a two-entry list of byte ranges
**pointing directly into the open files**, records the recipient and the completion callback,
hands the list to the transfer layer, and marks itself busy.

**Invariants** — the two payloads are transferred as one stream in the order given: logo, then
rules. The receiver must split them the same way, so the *order and count* are part of the
protocol even though neither is sent.

**Notes** — the byte-range list is built in stack-scoped storage, which is safe only because the
transfer layer copies what it needs from the list — not the payloads, which it borrows for the
duration. That is two different lifetime rules for two arguments of one call, and it is the sort of
thing a rebuild removes by passing owned handles.

## `upload_server_info_callback`

**Contract** — the transfer layer's status sink. Progress reports return without changing state.
Every terminal status — complete, rejected by the peer, aborted — marks the uploader idle and fires
the completion callback.

**Notes** — an abort by the local user is treated as a **fatal error**, which is inconsistent with
the other two terminal states and is wrong: there is no way for a server operator to abort a logo
upload, so the branch is unreachable, and if it became reachable it would take the server down. A
rebuild should treat it exactly like a peer rejection.

Rejection by the peer is not an error at all — a client may decline the transfer — and it still
completes the join. That is the right shape: **the info transfer is optional and never blocks a
player from entering**.

## `terminate_upload`

**Contract** — abort an upload in flight: tell the transfer layer to stop the transfer addressed by
this (recipient, sender) pair, mark idle, and fire the completion callback. Invoked from the
destructor when an uploader is torn down while busy.

**Invariants** — firing the completion callback on the abort path is what stops a client from being
stranded when the server shuts down mid-join. It is the same one-signal-per-client guarantee stated
above, held at the least convenient moment.
