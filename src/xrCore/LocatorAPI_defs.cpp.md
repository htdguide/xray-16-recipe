# src/xrCore/LocatorAPI_defs.cpp

> Resolves a logical root plus a relative name into one physical path, and matches a name against a wildcard mask.

**Needs** — [`LocatorAPI_defs.h`](LocatorAPI_defs.h.md) · [`LocatorAPI.h`](LocatorAPI.h.md) · [`xrstring.h`](xrstring.h.md) · [`_flags.h`](_flags.h.md)
**Used by** — reached through its declarations in [`LocatorAPI_defs.h`](LocatorAPI_defs.h.md); callers name that, not this file.
**Tier floor** — T2: string normalization and a matcher. It touches the real filesystem only to probe for a case-exact file on case-sensitive systems.

## Purpose

The virtual filesystem is addressed by *aliases* — `$game_data$`, `$level$`, `$app_data_root$` and about twenty more — and every lookup in the engine goes through one of them. This file owns the two-step translation: an alias is bound once to a root prefix plus an optional sub-path, and thereafter every name handed to that alias is normalized and concatenated onto the prefix. It also owns the listing entry type and the wildcard matcher that directory enumerations filter with.

## State

```text
RECORD LogicalRoot                  # one alias binding
  root        : text                # physical prefix, lowercased, separator-terminated
  add         : optional<text>      # sub-path appended to root, lowercased
  path        : text                # invariant: == root + add, separator-terminated, lowercased
  default_ext : optional<text>      # e.g. ".ltx"; used by the editors' save dialogs
  filter       : optional<text>     # human-readable file-dialog caption; editors only
  flags       : { recurse, notify, needs_rescan }

RECORD ListingEntry
  name        : text                # invariant: lowercased at construction, never after
  size        : int
  modified    : timestamp
  flags       : { is_directory, lives_in_archive }
```

**Invariant** — `path` is always `root` concatenated with `add` and always ends with the platform separator. Every mutation of either half recomputes `path`; nothing may write `path` directly. This is what lets a lookup be a bare concatenation with no separator bookkeeping at the call site.

**Invariant** — every stored string in both records is lowercased. This is the *storage* half of the case rule in §4 of the system requirements: the shipped game data references files with inconsistent case, so the engine folds case on the way in and never again. The fold is ASCII-only by construction, and must stay that way — a locale-aware fold changes which files match.

## `LogicalRoot` construction and rebinding

**Contract** — construction takes a root prefix, an optional sub-path, an optional default extension, an optional dialog caption and the flag set; it composes and stores the normalized `path`. Rebinding the sub-path, or rebinding the root, recomposes `path` from the two halves. All of them are total: an absent piece contributes nothing rather than failing.

**Invariants** — after any of the three, `path` ends in a separator and is lowercase. Path separators are normalized to the platform's own on composition, so game data authored with one convention resolves on a host using the other.

**Notes** — the editor build additionally creates the directory chain on disk at construction, because editors expect to be able to save into a root the moment it exists. A shipping rebuild should not do this: creating directories as a side effect of *describing* one is how a read-only installation acquires stray empty folders.

## `resolve` (the alias-plus-name lookup)

**Contract** — given a relative name, produces the physical path. On a case-sensitive host it first tries the name *as written*, concatenated onto the root, and returns that if such a file exists either on disk or inside a mounted archive. Only if that probe fails does it lowercase the name and concatenate. On a case-insensitive host the probe is skipped and the lowercase path is produced directly.

```text
FUNCTION resolve(root: LogicalRoot, name: text) -> text
  IF host filesystem is case-sensitive
    candidate <- root.path + name          # exactly as the caller spelled it
    IF exists(candidate) IN external_filesystem OR exists(candidate) IN mounted_archives
      RETURN candidate
  RETURN lowercase(root.path + lowercase(name))
```

**Notes** — the as-written probe is the whole reason this function is not a one-line concatenation. Archive directories store lowercased paths, so the lowercase branch always finds an archived file; loose files unpacked by a user on a case-sensitive host keep whatever case the unpacker produced, and only the as-written branch finds those. The cost is one extra existence check per lookup on those hosts. The original marks the compile-time host test as something that should become a runtime probe of the actual mount, which is the right call — the same binary can see both kinds of filesystem.

## `mark_for_rescan`

**Contract** — sets the needs-rescan flag on this root and on the filesystem as a whole, so the next enumeration re-reads the directory instead of answering from the cached listing. Does no work itself.

## `PatternMatch`

**Contract** — matches a name against a mask in which `?` matches exactly one character and `*` matches any run, including empty. Returns whether the whole name is consumed. Case is not folded here; callers pass names that are already lowercase. No allocation, no recursion.

```text
FUNCTION pattern_match(s: text, mask: text) -> bool
  # Phase 1: the literal prefix before the first star must match position for position.
  WHILE s not exhausted AND current mask char is not '*'
    IF mask char is neither '?' nor equal to the s char
      RETURN false
    advance both

  # Phase 2: greedy scan with one backtrack point.
  star_mask <- none      # mask position just after the last '*' seen
  star_s    <- none      # the s position that '*' is currently assumed to end at
  LOOP
    IF s exhausted
      RETURN true only if the rest of mask is all '*'
    IF current mask char is '*'
      advance mask
      IF mask exhausted
        RETURN true                    # trailing star swallows everything
      star_mask <- mask position
      star_s    <- s position + 1
      CONTINUE
    IF mask char is '?' OR equal to the s char
      advance both
      CONTINUE
    # mismatch: let the last star eat one more character and retry
    mask <- star_mask
    s    <- star_s
    star_s <- star_s + 1
```

**Notes** — the single backtrack point is what makes this linear in the common case and quadratic only in the worst; it is the standard iterative glob and there is nothing engine-specific about it. Note the asymmetry with phase 1: once a star has been seen there is always somewhere to back off to, so the loop never needs a failure exit other than exhausting the mask.
