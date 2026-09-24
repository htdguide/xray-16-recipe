# src/xrGame/InfoPortion.cpp

> Reads one info portion's authored definition out of the XML pool, and serializes the record of having received one.

**Needs** — [`InfoPortion.h`](InfoPortion.h.md) · [`xml_str_id_loader.h`](../xrServerEntities/xml_str_id_loader.h.md) · [`PhraseScript.h`](PhraseScript.h.md) · [`GameObject.h`](GameObject.h.md) · [`xrServerEntities/InfoPortionDefs.h`](../xrServerEntities/InfoPortionDefs.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — reached through its declarations in [`InfoPortion.h`](InfoPortion.h.md); callers name that, not this file.
**Tier floor** — T3: XML traversal and string identifiers

## Purpose

An **info portion** is the game's unit of knowledge: a named fact an inventory owner either
has or does not have. Almost every piece of story gating in the shipped games is a test for
an info portion, and almost every consequence — a dialog unlocking, an encyclopedia article
appearing, a task starting — is authored as a *side effect of receiving one*.

This file does two separable jobs. It fills a portion's shared definition from the XML pool
on first use, and it gives the "I have this portion, at this time" record its on-disk form.
The definition is shared per identifier across every owner who knows the fact, because the
authored payload is immutable and the games declare thousands of portions.

## State

The definition a portion identifier resolves to, loaded once and shared:

```text
RECORD InfoPortionDefinition
  dialog_names     : list<text>   # dialogs this fact makes available
  articles         : list<text>   # encyclopedia articles it reveals
  articles_disable : list<text>   # articles it retracts, so one can replace another
  game_tasks       : list<text>   # tasks it starts
  script_actions   : DialogScriptHelper   # what runs on the receiver when it lands
  disable_info     : list<text>   # other portions this one erases on arrival
```

The receipt record, one per owner per known fact:

```text
RECORD InfoReceipt
  info_id      : text
  receive_time : int (64-bit game time)
```

The pair is the whole model: a definition says what receiving the fact *does*, and a receipt
says that an owner has it and when. `disable_info` makes knowledge non-monotonic — a later
fact can unmake an earlier one — which is why the receipt store is a list that gets erased
from rather than a set that only grows.

## `CInfoPortion::load_shared`

**Contract** — resolves the portion's identifier against the preloaded index of XML files
and fills the shared definition from the matching node. Called once per identifier, on first
use; every later owner of the same fact reuses the result. An identifier with no node is not
an error in every game: for the two older titles it logs a warning and leaves the definition
empty, so a mod referencing a portion the data does not declare still runs. For the newest
title it is a hard failure.

**Invariants** — after a successful load every list is exactly what the node declares,
replacing anything previously there; a node with no children of a kind leaves that list
empty rather than absent.

```text
FUNCTION load_shared(info_id)
  entry = index.lookup(info_id)
  IF entry IS none
    IF game IS shadow_of_chernobyl OR game IS clear_sky
      log "attempt to use non-existent info portion"
      RETURN                       # tolerated: older data is known to be incomplete
    FAIL WITH missing_info_portion

  node = entry.document.node_at(entry.position, tag "info_portion")
  definition.dialog_names     = all child text of node, tag "dialog"
  definition.disable_info     = all child text of node, tag "disable"
  definition.script_actions   = load script helper from node
  definition.articles         = all child text of node, tag "article"
  definition.articles_disable = all child text of node, tag "article_disable"
  definition.game_tasks       = all child text of node, tag "task"
```

**Notes** — the article and task readers reject an empty attribute outright while the dialog
and disable readers silently accept one. The asymmetry is not a decision; it is two editing
passes. A rebuild should reject an empty identifier everywhere.

The node is addressed by a recorded *position within the file* rather than by searching for
the identifier, because the index pass already walked every file once and kept the offset.
That is what makes several thousand portions across many documents affordable.

## `CInfoPortion::InitXmlIdToIndex`

**Contract** — supplies the index builder with the two things it needs before it can scan:
the element name that denotes a portion, and the list of files to scan, read from
configuration. Both are set only if still unset, so a game that overrides them keeps its
override.

## `INFO_DATA::load` / `INFO_DATA::save`

**Contract** — the receipt's wire form: identifier then receive time, in that order, with no
framing of its own. It appears inside an inventory owner's saved state and inside the
network update that grants a fact, so the field order is frozen.

## `_destroy_item_data_vector_cont`

**Contract** — releases the XML documents behind an index at shutdown. The index holds many
entries per document, so the routine collects the *distinct* documents before releasing
them; releasing per entry would release the same document many times.

**Notes** — deduplication here is a consequence of an index that stores a document handle on
every entry instead of a document table with entries pointing into it. A rebuild that owns
documents in one table and refers to them by index needs none of this.
