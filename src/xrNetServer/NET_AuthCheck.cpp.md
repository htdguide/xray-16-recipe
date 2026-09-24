# src/xrNetServer/NET_AuthCheck.cpp

> Names the game files whose contents must match between server and client, and the ones that
> are allowed to differ.

**Needs** — [`NET_AuthCheck.h`](NET_AuthCheck.h.md) · [`xrCore/LocatorAPI.h`](../xrCore/LocatorAPI.h.md)
**Used by** — reached through its declarations in [`NET_AuthCheck.h`](NET_AuthCheck.h.md); callers name that, not this file.
**Tier floor** — T3: two lists of logical paths and a prefix test. Nothing here is
performance- or layout-sensitive.

## Purpose

A multiplayer server refuses a client whose game data differs from its own. That comparison is
one 64-bit number — see below — and this file decides which files go into it. It is a policy
file dressed as code: every line is a judgement about whether a particular kind of difference
between two installations is cheating or merely localization.

## State

Stateless — it produces two lists on demand.

## `fill_auth_check_params`

**Contract** — fills two lists of logical paths: the files that must match, and the files
that are exempt even though they sit inside a checked tree. Both are built from the virtual
filesystem's logical roots, so they resolve differently per installation but name the same
things.

```text
CHECKED - a difference here means the two installations are not the same game
  the entire configuration tree          # weapon damage, AI parameters, everything tunable
  the entire script tree                 # game logic
  the entire shader tree                 # a modified shader can see through walls
  the material and weapon sound trees    # a silenced weapon is an advantage
  five specific crosshair textures       # a hand-drawn scope reticle is an advantage
  the engine's own modules               # collision, core, sound, particles, both renderers,
                                         # the material system, the transport, the
                                         # matchmaking module, and the physics library
                                         # unless it is statically linked

EXEMPT - a difference here is legitimate and must not fail the check
  the writable data directory            # saves, settings, logs - never identical
  the localization selector              # which language is installed
  the font table                         # follows the language
  the item table                         # ships differing between editions
  the text, gameplay, UI and script sub-trees of the configuration
  one script-sound table and one named script file
```

**Notes** — the exemptions carve *out of* the checked configuration tree, so the two lists are
not independent: the ignore list is applied first and the check list second. Four of the five
exempt configuration sub-trees are exempt for the same reason — they are per-language — and
the fifth, the gameplay tables, is exempt because those tables ship differing between
editions of the game.

The two file-specific exemptions are a localized sound table and a single named script. Those
are not policy; they are two known files that differed between shipped builds and would
otherwise have made every server unjoinable. A rebuild starts from a clean set and will not
need them.

The physics library is checked only when it is a separate module; in the fully-linked shipping
configuration it is inside the executable and the executable is not checked at all. The
executable's exclusion is explicit in the source — commented out, not forgotten — which means
a client running a *modified engine* passes the check. The check is about data, not code, and
calling it an anti-cheat measure overstates it.

## `allow_to_include_path`

**Contract** — takes the exempt list and a path, and answers whether that path may be
included in the checksum. Compares by prefix: the path is exempt if any list entry is a prefix
of it.

**Notes** — the same predicate has a second, unrelated caller: the configuration parser
consults it when deciding whether to follow an include directive, so that a server's
authenticated configuration set and its *loaded* configuration set are the same files. Two
uses of one list, and the coupling is invisible from either side. A rebuild should name the
list once and hand it to both.

## How the number is actually formed

Not in this file — the combination lives with the virtual filesystem — but a rebuilder needs
it here, because nothing else states it:

```text
FUNCTION authentication_code(ignore, check) -> int (64-bit)
  code = crc32(the merged configuration set, serialized back to text)
  FOR EACH file IN the mounted filesystem
    IF any ignore entry occurs in the file's name THEN CONTINUE
    IF any check entry occurs in the file's name AND the file is non-empty THEN
      code = code XOR crc32(contents of file)
  RETURN code
```

Three honest observations. Combining with exclusive-or makes the result independent of
enumeration order, which is necessary — two installations enumerate their archives
differently — but it also means a file whose checksum appears twice cancels out, and two files
that swap contents are indistinguishable. Matching is by *substring*, not prefix, on both
lists, so a path containing `ui` anywhere is treated as checked. And the base value is the
checksum of the *resolved* configuration set rather than of its files, which is the right
choice: it compares what the two engines will actually use, after inheritance and includes.

The comparison itself happens in the game layer: the server challenges, the client answers
with its code, and a mismatch is a refusal with the reason "data verification failed" — unless
an operator has set the variable that ignores version mismatches, which exists precisely
because this check is too blunt to live with during development.
