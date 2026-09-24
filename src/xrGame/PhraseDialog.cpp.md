# src/xrGame/PhraseDialog.cpp

> One conversation: a graph of phrases, two speakers taking strict turns, and the rules that decide which replies are offered next.

**Needs** — [`PhraseDialog.h`](PhraseDialog.h.md) · [`Phrase.h`](Phrase.h.md) · [`PhraseDialogManager.h`](PhraseDialogManager.h.md) · [`PhraseScript.h`](PhraseScript.h.md) · [`GameObject.h`](GameObject.h.md) · [`Actor.h`](Actor.h.md) · [`xrAICore/Navigation/graph_abstract.h`](../xrAICore/Navigation/graph_abstract.h.md) · [`xml_str_id_loader.h`](../xrServerEntities/xml_str_id_loader.h.md) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — reached through its declarations in [`PhraseDialog.h`](PhraseDialog.h.md); callers name that, not this file.
**Tier floor** — T3: graph walking and script dispatch

## Purpose

A dialog is an authored directed graph whose vertices are phrases and whose edges are "this
may be said in reply to that". This file loads that graph from XML and runs one traversal of
it between two speakers.

The two halves are deliberately separate and share nothing but the graph: the **definition**
is immutable, loaded once per dialog identifier and shared by every conversation using it;
the **traversal** is per-conversation and holds only the current position, the currently
offered replies, and who speaks next. That split is what lets a hundred creatures hold the
same authored conversation without a hundred copies of it.

The graph is *not* a tree even though it is loaded by a recursive descent: an edge may point
back at an already-created phrase, which is how loops and shared endings are authored.

## State

The shared definition:

```text
RECORD PhraseDialogDefinition
  caption        : text                      # menu label; empty means "use phrase 0's text"
  phrase_graph   : graph of Phrase, keyed by phrase id, edges weighted (weight unused)
  script_helper  : DialogScriptHelper        # the whole dialog's start preconditions
  priority       : int                       # sort order in the player's dialog list,
                                             # descending; may be negative
```

The per-conversation traversal:

```text
RECORD PhraseDialog
  dialog_id        : text
  said_phrase_id   : text          # "" before anything is said
  finished         : bool
  available        : list<Phrase>  # what the listener may say now, sorted by goodwill desc
  speaker_first    : DialogManager
  speaker_second   : DialogManager
  first_is_speaking: bool          # whose turn it is to say something
```

**Invariants**
- Exactly two speakers, both set or neither; a conversation with one speaker is not
  initializable.
- `available` is non-empty whenever the conversation is not finished. An authored graph that
  can reach a state with outgoing edges but no passing precondition is a data bug and is
  raised as one; the system has no "nothing to say" state short of finished.
- `available` is sorted by goodwill descending, so the most demanding reply is offered first.
- Turn strictly alternates: saying a phrase flips `first_is_speaking` before anything else
  happens.

## `Init`

**Contract** — binds a loaded definition to two speakers and positions the traversal at the
entry phrase. Refuses if already bound. Seeds the offered list with the entry phrase alone —
*without* testing its precondition, because whether this conversation may start at all was
already decided by `Precondition` on the whole dialog. The first speaker opens.

**Invariants** — the definition must contain a phrase with identifier `"0"`; a dialog
without one is a data error, not an empty conversation.

## `SayPhrase`

**Contract** — the entire step of the conversation. Records the phrase as said, flips the
turn, runs the said phrase's script effect, recomputes what the *other* speaker may now say,
and hands the conversation to that speaker. Returns whether the conversation continues.
Takes the conversation by shared handle rather than by plain reference because the script
effect it fires may itself end and release the conversation — the routine must survive its
own side effect.

```text
FUNCTION say_phrase(dialog, phrase_id) -> bool
  REQUIRE dialog.is_inited
  dialog.said_phrase_id = phrase_id
  was_first = dialog.first_is_speaking
  dialog.first_is_speaking = NOT dialog.first_is_speaking   # turn flips first

  speaker = the one whose turn it just was
  listener = the other one
  vertex = dialog.definition.phrase_graph.vertex(phrase_id)

  # effect of having said it: speaker, then listener
  vertex.phrase.script_helper.action(speaker, listener, dialog.dialog_id, phrase_id)

  dialog.available = empty
  IF vertex has no outgoing edges
    dialog.finished = true
  ELSE
    FOR EACH edge IN vertex.edges
      next = graph.vertex(edge.target)
      # precondition is asked from the LISTENER's point of view: arguments swap,
      # because the listener is the one who would say the reply
      IF next.phrase.script_helper.precondition(listener, speaker,
                                                dialog.dialog_id, phrase_id, next.id)
        dialog.available.append(next.phrase)
    FAIL WITH no_available_phrase IF dialog.available IS empty
    sort dialog.available by goodwill_level descending

  listener.receive_phrase(dialog)      # may end and release the dialog
  RETURN dialog still alive AND NOT dialog.finished
```

**Notes** — the argument swap on the precondition call is the load-bearing detail. An
authored precondition is written as "can *I* say this to *him*", so the pair handed to it
must be (the one who would speak the reply, the one who would hear it) — which is the
reverse of the pair handed to the effect of the phrase just said.

The final return re-checks that the conversation still exists before reading its finished
flag, and reports "continue" when it does not. The caller treats a released conversation as
still running and discovers the truth through its own handle going empty; a rebuild would do
better to report termination.

The finalizer flag on a phrase is read by the dialog *manager*, not here — this routine ends
a conversation only when the graph runs out of edges.

## `GetPhraseText`

**Contract** — resolves a phrase's display text at the moment it is shown, with three
sources tried in order of specificity. Mutates the phrase's cached script text as a side
effect, so it is not safe to call concurrently on the same shared definition.

```text
FUNCTION get_phrase_text(phrase_id, current_speaking = true) -> text
  phrase = graph.vertex(phrase_id).phrase
  # when asked for a phrase already said, no speakers are passed: the text is
  # wanted for the log, not for a live exchange
  IF NOT current_speaking
    s1 = none; s2 = none
  ELSE
    s1 = first_speaker; s2 = second_speaker

  # the script is always told about the NON-player participant, whichever side he is on
  subject = IF s1 is the actor THEN s2 ELSE s1

  IF phrase.script_text_id IS NOT empty
    fn = script function named phrase.script_text_id
    FAIL WITH missing_function IF fn not found
    phrase.script_text_val = fn(subject, dialog_id, phrase_id)
    RETURN phrase.script_text_val

  RETURN phrase.script_helper.script_text(phrase.text, s1, s2, dialog_id, phrase_id)
```

**Notes** — passing the non-player participant rather than the speaker is what lets one
authored line serve both directions of a conversation: the script that fills in a name is
always asking about the other character, never about the player.

## `DialogCaption`

**Contract** — the label the player's dialog list shows. The authored caption if there is
one; otherwise the text of the entry phrase, resolved through the full three-source path
above. So a dialog with no caption costs a script call every time the list is drawn.

## `Priority`

**Contract** — the sort key for the player's list of available dialogs, higher first.
Negative values are meaningful and used to push a dialog to the bottom.

## `Precondition`

**Contract** — whether this conversation may be started at all, between these two. Runs the
whole dialog's authored predicate with an empty phrase identifier, since no phrase is in
play yet. The per-phrase preconditions are a separate, later gate.

## `Load` / `load_shared`

**Contract** — binds an identifier and fills the shared definition from the XML pool on
first use, by position rather than by search. Two authoring modes are supported and they are
mutually exclusive:

```text
FUNCTION load_shared()
  node = xml pool entry for dialog_id, at its recorded position
  priority = attribute "priority", default 0
  caption  = child "caption", may be absent
  load the dialog's own script preconditions from node
  clear the phrase graph

  phrase_list = child "phrase_list"
  IF phrase_list IS none
    # scripted dialog: a named function builds the graph by calling the
    # add-phrase surface directly. Used for dialogs whose shape depends on world state.
    fn = script function named by attribute "init_func"
    FAIL WITH missing_function IF fn not found
    fn(this)
    RETURN

  FAIL WITH empty_dialog IF phrase_list has no "phrase" children
  # (debug builds additionally reject duplicate phrase ids here)
  entry = the "phrase" child whose id is "0"
  add_phrase_recursive(entry, id "0", predecessor "")
```

**Notes** — the recursive load starts from the phrase identified `"0"` and reaches only
phrases connected to it. A phrase authored into the file but unreachable from the entry is
silently never loaded, which is why an orphaned phrase in the shipped data produces no
diagnostic.

## `AddPhrase` (recursive XML form)

**Contract** — creates one phrase from its node, then descends into every phrase it names as
a reply, linking each back to itself. Because vertex creation is idempotent on identifier,
an edge into an already-built phrase links without rebuilding, and a cycle terminates.

```text
FUNCTION add_phrase_recursive(node, phrase_id, prev_phrase_id)
  text     = child "text", default empty
  goodwill = child "goodwill", default -10000    # effectively "anyone may say this"
  phrase = add_phrase(text, phrase_id, prev_phrase_id, goodwill)
  IF phrase IS none
    RETURN                    # already built on another path; its children too

  phrase.is_finalizer  = (child "is_final" == 1)
  phrase.script_text_id = child "script_text", default empty
  load phrase script helper from node

  FOR EACH child "next" AS next_id
    next_node = the "phrase" child with that id
    FAIL WITH dangling_reply IF next_node IS none
    add_phrase_recursive(next_node, next_id, phrase_id)
```

**Notes** — the default goodwill of -10000 is a sentinel meaning "no standing requirement".
It is a number rather than an absent value because the sort key must exist for every phrase;
a rebuild with an optional threshold should sort absent as lowest.

Returning nothing for an already-built phrase is what prunes the recursion, but it also
means a phrase's properties come from *whichever* path reached it first. Two authored copies
of the same identifier with different bodies resolve to the first one loaded.

## `AddPhrase` (direct form)

**Contract** — creates or finds a phrase by identifier, and optionally links a predecessor
to it. Returns the new phrase, or nothing when one already existed under that identifier.
This is also the surface a script-built dialog uses.

**Notes** — duplicate identifiers carrying identical text are common in the shipped data and
are accepted silently; only a duplicate with *different* text warns, and only outside release
builds. That tolerance is a concession to the shipped files, not a design choice.

An edge is added with weight zero and no traversal ever reads the weight. The graph structure
is borrowed from the navigation code, which needs weights; dialogs do not.

## `GetPhrase`

**Contract** — the phrase under an identifier. Absence is a hard failure, not an empty
result: every identifier reaching this point came from an edge in the same graph.

## `allIsDummy`

**Contract** — true when every currently offered reply has no text from any source. The
conversation UI uses this to distinguish "the other party is still talking" from "the player
is being shown an empty menu", and drives the exchange forward without player input.

## `CurrentSpeaker` / `OtherSpeaker` / `LastSpeaker` / `OurPartner` / `IsWeSpeaking`

**Contract** — the turn bookkeeping, all derived from the two speaker handles and one
boolean. `OurPartner` answers "who is the other one" for a caller that knows only itself,
and returns the first speaker when handed something that is neither — a rebuild should
require membership.

## `Reset`

**Contract** — declared as the reinitialization hook and does nothing. Nothing in the tree
depends on it having an effect; a conversation is rebuilt rather than reset.

## `InitXmlIdToIndex`

**Contract** — names the element that denotes a dialog and the file list to scan, the latter
read from configuration. Set only if unset.
