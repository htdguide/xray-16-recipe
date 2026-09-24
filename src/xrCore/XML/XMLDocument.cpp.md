# src/xrCore/XML/XMLDocument.cpp

> Flattens a file's include directives into one text image, parses it, and then answers colon-separated path queries against the tree with a default for every miss.

**Needs** — [`XMLDocument.hpp`](XMLDocument.hpp.md) · [`tinyxml.h`](tinyxml.h.md) · [`FS.h`](../FS.h.md) · [`LocatorAPI.h`](../LocatorAPI.h.md) · [`xrstring.h`](../xrstring.h.md)
**Used by** — [`XMLDocument.hpp`](XMLDocument.hpp.md)
**Tier floor** — T2: line-oriented preprocessing, a tree walk, and string-keyed lookup.

## Purpose

Two jobs that happen to share a file. The first is a **preprocessor**: the shipped interface layouts are split across dozens of files stitched together with an include directive that XML itself has no notion of, so before anything is parsed the file is read line by line and every include is replaced by the contents of the named file, recursively. The second is a **query surface**: the interface code addresses elements by a path like `window:caption:text` with an index for repeated siblings, and every read has a default so that an absent element is a designed-for case rather than an error.

The parse itself belongs to the vendored parser next door; this file's contribution to it is deciding what counts as a fatal error.

## State

```text
RECORD XMLDocument
  file_name  : text                  # for diagnostics
  document   : ParsedTree            # the whole tree, owned
  root       : optional<Node>        # the first element child of the document
  local_root : optional<Node>        # if set, path lookups start here instead
  admit_missing_end_tag : bool
  scratch    : list<text>            # reused by the duplicate-attribute check
```

**Invariant** — `root` is the first *element*, not the first node, so a leading declaration or comment does not become the root.

**Invariant** — when a local root is set, every path lookup and every node count that does not name a starting node begins there. This is how one loaded file serves many screens: the interface code parks the local root on a screen's sub-tree and then reads relative paths.

## The include directive

The directive is line-oriented and lives *outside* XML's grammar, so it must be resolved before parsing:

```text
#include "some/path/file.xml"
```

Leading blanks are allowed before the directive and between it and the quoted name. The name may not be empty and may not exceed 1024 characters. Anything else on a line that begins with the directive is an error; a line that does not begin with it is copied through verbatim.

```text
FUNCTION flatten(path_root, reader, out, depth)
  IF depth >= 128
    FAIL WITH IncludeTooDeep            # guards against a cycle
  WHILE not at end of reader
    line <- read one line               # a line of 4096 bytes or more is an error
    IF line is an include directive
      name <- the quoted name
      inner <- resolve_include(path_root, name)
      IF inner is none
        FAIL WITH MissingInclude(name)
      flatten(path_root, inner, out, depth + 1)
    ELSE
      append line to out
  # the caller appends a zero terminator once the whole tree is flattened
```

**Invariants** — the depth cap of 128 is the only cycle protection; a file that includes itself is caught by it, not by a visited set. The line cap of 4096 bytes is a hard limit on the shipped data and a parse failure if exceeded.

### Resolving an include name

This is where the interface-directory override in [`XMLDocument.hpp`](XMLDocument.hpp.md) does its work. Three candidates are tried in order, and the first that opens wins:

```text
FUNCTION resolve_include(path_root, name) -> optional<Reader>
  # 1. name already starts with the CURRENT interface directory:
  #    strip its first component, let the subclass rewrite the remainder,
  #    and re-root it under the current interface directory
  # 2. name starts with the DEFAULT interface directory:
  #    same, but re-rooted under the CURRENT one — this is the override,
  #    letting a localization redirect a default-named include
  # 3. name starts with the DEFAULT interface directory:
  #    re-rooted under the DEFAULT one — the fallback when the override
  #    has no file of that name
  # 4. otherwise: resolve the name as given, relative to path_root
```

**Notes** — the subclass rewrite hook is consulted in every one of the first three candidates, which is how a language variant substitutes a translated layout file without any of the including files changing. The fourth candidate is the ordinary case and the only one most files take.

## `Load` and `Set`

**Contract** — `Load` opens the file, flattens it into an in-memory image, terminates it, and hands it to `Set`. A missing file is fatal in the default mode and a returned false otherwise — every caller that can cope with an absent layout passes the non-fatal flag. The three forms differ only in path composition: one takes the whole relative path, one composes a directory and a filename, and one composes two candidate directories and takes the first that loads.

`Set` parses text directly. It records the error description, establishes the root as the first element, and applies the one error exemption below.

```text
FUNCTION set(text, fatal) -> bool
  parse(text)
  IF the parse reported an error
    forgivable <- admit_missing_end_tag AND the error is exactly "reading end tag"
    IF fatal AND NOT forgivable
      FAIL WITH ParseError(description, file_name)
    IF NOT forgivable
      RETURN false
  root <- first element child
  RETURN true
```

**Notes** — the missing-end-tag exemption exists because some shipped files are genuinely malformed in exactly that way and the original engine tolerated them. The exemption is *narrow by design* — one specific error identifier, opted into per document — and a rebuild must keep it that narrow: widening it to "ignore parse errors" would silently truncate files that are wrong for other reasons.

Note that a forgiven document still returns success with whatever tree the parser managed to build, which is the prefix up to the fault. Callers get a partial tree and do not know it.

Applying the subclass filename rewrite happens *before* path composition in the two composing forms, so the rewrite sees the bare filename and not the directory. That is the contract the localization layer relies on.

## `NavigateToNode` — the path language

**Contract** — resolves a path against a starting node and returns the node, or nothing. The path is components separated by colons. The index selects among *same-named siblings at the first level only*; deeper levels always take the first match. Absent a starting node, the local root is used if set, otherwise the document root. A path longer than 200 characters is a checked error.

```text
FUNCTION navigate(start, path, index) -> optional<Node>
  tokens <- path split on ':'
  node <- start.first_child_named(tokens[0])
  REPEAT index times
    node <- node.next_sibling_named(tokens[0])     # stops early at none
  FOR EACH remaining token
    IF node is none
      BREAK
    node <- node.first_child_named(token)
  RETURN node
```

**Invariants** — the index applies only to the first component. That asymmetry is real and is what the interface code expects: `tab:button` with index 3 means the fourth `tab`, then its first `button` — never the fourth `button`.

**Notes** — the path is copied into scratch storage and split in place, which is why the length is bounded. The original carries a comment marking the tokenizer as thread-safe, which is a correction of an earlier version that used a shared tokenizer state; a rebuild simply splits the string.

## The readers

**Contract** — each reader resolves its node (by path, by path from a node, or directly), extracts a value, and returns a caller-supplied default if the node is missing, has no text child, or the child is not text. The numeric readers ask for the text with a *null* default and substitute the numeric default when they get nothing back, so a node whose text is genuinely the string "0" is distinguishable from an absent node. Numeric conversion is permissive: leading whitespace is skipped, trailing garbage is ignored, an unparseable value yields zero rather than the default.

**Notes** — that last point is the trap. `ReadInt` on a node containing `abc` returns 0, not the default. A rebuild should decide whether that is acceptable; the shipped data does not exercise it.

A node's value is the text of its **first child**, so an element with a leading comment reads as empty. Every shipped file avoids this by convention.

## `GetNodesNum`

**Contract** — counts children of a node carrying a given tag; with no tag, counts all children. A flag decides whether comment nodes are counted. Returns zero for an absent node. The path-taking form falls back to the root if the path does not resolve — so an unresolvable path counts the root's children rather than reporting nothing, which is a silent surprise a rebuild should reconsider.

**Notes** — the comment flag defaults to *including* comments, which means the common `GetNodesNum` result is "the number of children including the ones the author wrote as documentation". Callers that iterate `0 .. count-1` and navigate by index therefore encounter comment nodes. It works because navigation by name skips them; it would break immediately for an untagged count.

## `SearchForAttribute` and `NavigateToNodeWithAttribute`

**Contract** — both find an element with a given tag whose named attribute matches a value exactly, by byte comparison with no case folding. The navigating form searches only the direct children of the root; the searching form descends recursively along same-tag paths. Both return nothing on no match.

**Notes** — the recursive form only descends into children that *also* carry the search tag, so it finds nested occurrences of one repeated element and will not find a match buried under differently-named ancestors. That is narrower than "search the subtree" and is what the interface's repeated-element lookups need.

The navigating form does its search by counting the children and then reading each one's attribute by index, which is quadratic in the child count because each index restarts the sibling walk. With the dozens of elements the shipped files hold this never mattered; a rebuild should iterate siblings once.

## `CheckUniqueAttrib`

**Contract** — walks the same-named children of a node, collects one named attribute's value from each, and returns the first value seen twice, or nothing. Uses a member scratch list, cleared at both ends, so it is not reentrant and not safe to call concurrently on one document. A data-validation tool, not a runtime path.

**Notes** — the comparison is linear over the values collected so far, which is fine for the sizes involved. Collecting an absent attribute pushes an empty entry, so two children both missing the attribute count as a duplicate — which is arguably the right answer for the check's purpose.
