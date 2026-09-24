# src/xrGameSpy/GameSpy_HTTP.cpp

> A single-slot file downloader with progress reporting: how the engine fetches a patch or
> a multiplayer level it does not have.

**Needs** — [`GameSpy_HTTP.h`](GameSpy_HTTP.h.md) · [Seam: Multiplayer matchmaking and accounts](../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)
**Used by** — [`GameSpy_HTTP.h`](GameSpy_HTTP.h.md)
**Tier floor** — T2: a request, a poll, a file written incrementally.

## Purpose

The online layer needs to fetch two kinds of file the game did not ship with: a patch, and
a multiplayer level the server is running and the client does not have. Both are plain
downloads to a path on disk with a progress bar. This file is that, and only that.

It belongs to this chapter for a historical reason rather than a structural one: it is a
general file fetcher that happens to come from the matchmaking vendor's library. A
rebuild has no reason to keep it here — this is the one file in the chapter that is
*completely replaceable by an ordinary HTTP client*, with no dead service behind it.

## State

```text
RECORD FileDownloader
  current : optional<RequestHandle>   # at most one download at a time
```

**Invariants** — one slot. Starting a second download replaces the handle and orphans the
first, which then completes into a context that no longer exists. A rebuild should either
queue or refuse.

## `DownloadFile`

**Contract** — starts a download from a URL to a path, non-blocking, reporting progress
and completion through two caller-supplied callbacks. Logs the URL and the destination.
If the request is refused at submission — a malformed URL, no socket — the completion
callback fires immediately with failure, synchronously, before the call returns.

```text
FUNCTION download(url : text, destination : text, on_done, on_progress)
  ctx <- (on_done, on_progress)
  current <- submit_download(url, destination, save_to_file, ctx)
  IF current indicates refusal
    on_done(false)
```

**Invariants** — the callback pair is bundled into a context whose lifetime is **the
calling stack frame**, and handed to the request as an opaque value. The request
completes on a later poll, by which time the frame is gone. Same defect as in
[`GameSpy_GCD_Server.cpp`](GameSpy_GCD_Server.cpp.md), and here it is reachable on any
download that does not fail at submission. A rebuild must own the callbacks for the
request's lifetime.

**Notes** — the three log lines (URL, destination, submission code) are unconditional,
including in a shipping build. The destination and URL of a patch download are not
sensitive; a rebuild should still make them conditional.

## `Think`

**Contract** — one poll per frame, driven from the facade. Advances the transfer,
delivering progress and completion. Does not block.

## `StopDownload`

**Contract** — cancels the in-flight request if there is one and clears the slot. The
completion callback still fires, with the cancellation result, which the completion
handler reports as a failure like any other.

## `StartUp` · `CleanUp` · construction · destruction

**Contract** — bring the transfer machinery up and down. Construction starts it and
clears the slot; destruction stops it. No failure path is exposed.

## The progress and completion handlers

**Contract** — the progress handler reports `(received, total)` and **only while the body
is being received and only when the total length is known**, so a response without a
declared length reports no progress at all and the bar stays where it was. The completion
handler maps the result onto a boolean, logging the failure's name when it is not success.

**Notes**

The failure names are held as a **positional table indexed by the result code** — nineteen
entries covering memory, buffer, URL parsing, host lookup, socket, connect, malformed
response, rejected, unauthorized, forbidden, not found, server error, local write, local
read, interrupted, too large, encryption, and cancellation. A result outside that range
reads past the table. The list is worth keeping as a *taxonomy*: it is a complete
statement of how this download can fail, and a rebuild's downloader should distinguish the
same categories — particularly *interrupted* (which is only detectable when the length was
known) and *too large* (a file beyond the size the transfer can address), both of which a
patch download genuinely hits.

**The null path.** Nothing about this file depends on the dead service; only the *URLs*
did. The patch-check path and the level-download path both take their URL from
configuration, so a rebuild that points them somewhere live keeps working downloads.
With no URL configured, the engine simply never starts a download.
