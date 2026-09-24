# src/xrServerEntities/script_rtoken_list.h

> An ordered list of interned names a script builds, for configuration reads constrained to a data-driven vocabulary.

**Needs** — [`script_rtoken_list_inline.h`](script_rtoken_list_inline.h.md) · [`script_rtoken_list_script.cpp`](script_rtoken_list_script.cpp.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`script_properties_list_helper.h`](script_properties_list_helper.h.md) · [`script_rtoken_list_inline.h`](script_rtoken_list_inline.h.md) · [`script_rtoken_list_script.cpp`](script_rtoken_list_script.cpp.md)
**Tier floor** — T3: an ordered list of strings with index-based access.

## Purpose

The configuration reader can read a value *as a token*: the text in the file is matched
against a vocabulary and the matching entry's number is stored. Some vocabularies are fixed
at build time; some are built at run time from data, and for those the vocabulary itself has
to come from somewhere. This is that somewhere, for the case where the entry's identity
**is** its name.

Its sibling [`script_token_list.h`](script_token_list.h.md) is the case where a name is
paired with an explicit number. The difference matters: here the number is the **position in
the list**, which means inserting or removing an entry renumbers everything after it. That
makes this form suitable only for vocabularies that are rebuilt whole and never persisted by
number — an editor drop-down, a property list's choices — and unsuitable for anything that
reaches a save.

## State

```text
RECORD ScriptRTokenList
  names : list<text>    # interned; position is the token's value
```

**Invariants** — positions are dense and start at zero. Nothing enforces uniqueness of a
name, so a duplicate is silently allowed and the first occurrence wins any lookup that walks
the list.

## the exported surface

`add` (append), `remove` (by index), `get` (by index), `size`, `clear`, and access to the
underlying list for the native side that consumes it.

**Invariants** — `remove` and `get` are **bounds-checked and silent**: an out-of-range index
is ignored on removal and answers nothing on read. That is deliberate — the caller is a
script, and a script indexing past the end of a list should not take the process down.

## Notes

**Names are interned** rather than copied, which is the only difference in storage from the
paired form and the reason for the separate type. A vocabulary of a few hundred entries
built repeatedly — once per property list the editor opens — would otherwise churn. Interned
storage also means the list can be handed to the property model as a vocabulary without
copying it again.
