# src/xrCore/XML/tinyxml.cpp

> The tree itself: linking children, walking siblings by name, and reading an element's attributes.

**Needs** — [`tinyxml.h`](tinyxml.h.md) · [`tinystr.h`](tinystr.h.md) · [`xrMemory.h`](../xrMemory.h.md)
**Used by** — [`tinyxml.h`](tinyxml.h.md)
**Tier floor** — T2: a doubly-linked tree with owned children.

## Purpose

Implements the node model declared in [`tinyxml.h`](tinyxml.h.md) — everything except the grammar, which lives in [`tinyxmlparser.cpp`](tinyxmlparser.cpp.md). A rebuild almost certainly replaces this whole file with its language's own tree or DOM; what follows is what the engine's callers actually rely on.

## State

Each node holds its type, its value string, a parent link, first and last child links, and previous and next sibling links. Children are owned: destroying a node destroys its subtree.

**Invariant** — a node's parent link and its position in the parent's child list always agree. Linking a child sets both ends; removing one clears both.

**Invariant** — a node belongs to exactly one tree. Linking a node that already has a parent is not supported; the library's insertion operations take a node by value and *copy* it, which is why an element constructed on the stack and inserted leaves no dangling reference.

## Tree operations

**Contract** — link a node as the last child; clear all children; find the first or last child, optionally filtered by value; iterate children with or without a filter; find the next or previous sibling, optionally filtered; find the first child or next sibling that is specifically an *element*, optionally filtered; reach the owning document by walking parents. All the finders return nothing rather than failing. None of them allocates except linking, which copies the node being inserted.

**Notes** — every "filtered by value" finder compares the node's value string for equality, which for an element is its tag name. That single comparison is what [`XMLDocument.cpp`](XMLDocument.cpp.md)'s whole path language is built from, and it is exact and case-sensitive. The shipped data is consistently lower-case; a rebuild that folds case will match files the original rejects.

The element-specific finders skip every non-element sibling, which is why navigating by name transparently steps over comments while counting by position does not.

## Attributes

**Contract** — an element's attributes are an ordered list searched linearly by name. Fetching returns the value text, or nothing. Fetching with conversion returns the text and additionally writes the converted number through a caller-supplied slot, leaving it untouched if the conversion fails. A separate query form reports success, absence or wrong-type as distinct outcomes.

**Invariants** — attribute order is preserved in the order parsed. Nothing in the engine depends on it; only output would.

**Notes** — the linear scan is fine at the handful of attributes real elements carry, and would not be at hundreds.

Numeric conversion uses the permissive text-to-number routines, so trailing garbage is ignored and an unparseable value yields zero *and reports success*. The query form is the one that reports a wrong type, and the engine does not use it — see the trap noted in [`XMLDocument.cpp`](XMLDocument.cpp.md).

## `GetText`

**Contract** — returns an element's first child's text if that child is a text node, otherwise nothing. Does not concatenate, does not descend.

**Notes** — this is the rule that makes an element's content readable only when text is its *immediate first* child. It is the single most consequential simplification in the library for this engine's data, and a rebuild must reproduce it exactly or elements whose authors interleaved comments will start returning content the original never returned.

## The visitor

**Contract** — a depth-first traversal calling enter and exit callbacks for documents and elements and a single visit callback for the leaf kinds; returning false from an enter callback prunes that subtree.

**Notes** — unused by the engine. A rebuild can omit it.

## The whitespace flag

**Contract** — a single process-wide flag, defaulting to on, deciding whether runs of whitespace in text are condensed to one space. Set before parsing; changing it mid-parse is undefined.

**Notes** — a global mutable parser setting is the kind of thing a rebuild should make a per-parse option. The value matters (see [`tinyxml.h`](tinyxml.h.md)); its being global does not.
