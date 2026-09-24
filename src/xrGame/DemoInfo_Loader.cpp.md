# src/xrGame/DemoInfo_Loader.cpp

> Reads the summary block out of a recorded multiplayer demo file and caches it by filename, so a browse screen can list many demos without re-reading any.

**Needs** — [`DemoInfo_Loader.h`](DemoInfo_Loader.h.md) · [`DemoInfo.h`](DemoInfo.h.md) · [`Level.h`](Level.h.md) · [`xrCore/stream_reader.h`](../xrCore/stream_reader.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: streamed read of a frozen file prefix; no layout decisions of its own

## Purpose

A demo file is a recorded multiplayer match: a header, a level name, a summary of the
match and its players, then the recorded stream. The demo-browsing screen wants only the
summary, for every file in the directory, and wants it repeatedly as the player scrolls.
This module reads just the prefix and memoizes the result.

The caching is one-way and lasts the lifetime of the loader: entries are never
invalidated, because a demo file on disk does not change while the browser is open. The
loader owns every summary it has read and frees them all when it is destroyed.

## State

```text
RECORD DemoInfoLoader
  cache : map<text, DemoInfo>    # keyed by the filename as given, owned entries
```

Invariant: the cache owns its values. Nothing hands a summary out with transferable
ownership — callers receive a read-only view whose lifetime is the loader's.

## `get_demofile_info`

**Contract** — returns the summary for a named demo file, reading it on first request and
serving the cached copy afterwards. Blocks on the first request for a given name. A file
that cannot be read or parsed is a hard failure, not a `none`: the browse screen is
expected to have listed only files that exist.

```text
FUNCTION get_demofile_info(name) -> DemoInfo
  IF name IN cache THEN RETURN cache[name]
  info = load_demofile(name)
  FAIL WITH "unreadable demo" IF info IS none
  cache[name] = info
  RETURN cache[name]
```

**Notes** — the key is the filename exactly as passed. Two spellings of the same file
would produce two cached copies; no caller varies the spelling.

## `load_demofile` *(private, but the file layout is the point)*

**Contract** — opens the named file from the logs root as a stream, skips the fixed-size
demo header and the level-name string that follow it, hands the reader to the summary
record to parse itself, then sorts the players. Closes the stream before returning. The
header and level name are read into throwaway storage *rather than seeked past*, because
the stream reader is forward-only and the header's size is the only thing that positions
the summary.

```text
FUNCTION load_demofile(name) -> optional<DemoInfo>
  reader = open_stream("$logs$", name)
  IF reader IS none THEN
    log("failed to open " + name)
    RETURN none
  skip a demo header record        # fixed size; contents unused here
  skip one length-prefixed string  # the level name
  info = DemoInfo.read_from(reader)
  info.sort_players(by team, then by spots descending)
  close(reader)
  RETURN info
```

**Invariants** — the prefix layout is shared with the demo *writer* and must match it
exactly; it is frozen against itself, not against foreign data.

## Player ordering

**Contract** — the sort the loader imposes before anyone sees the summary: players group
by team ascending, and within a team by their score descending. The browse screen and the
end-of-match screen both rely on the summary arriving already ordered, so the ordering is
part of the loader's contract rather than a display concern.

**Notes** — "spots" is the per-player score this comparator ranks by; see
[`DemoInfo.cpp`](DemoInfo.cpp.md) for what the recorded per-player fields mean.
