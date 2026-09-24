# src/xrServerEntities/script_rtoken_list_inline.h

> The interned-name list's bodies: append, bounds-checked remove and get, size, clear.

**Needs** — [`script_rtoken_list.h`](script_rtoken_list.h.md)
**Used by** — [`script_rtoken_list.h`](script_rtoken_list.h.md)
**Tier floor** — T3.

## Purpose

Carries the definitions split out of [`script_rtoken_list.h`](script_rtoken_list.h.md). The
only content worth stating is the one already stated there: both index-taking operations
check the bound and do nothing rather than fail, because the caller is script code. In a
rebuild this merges into the type.
