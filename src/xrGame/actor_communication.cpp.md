# src/xrGame/actor_communication.cpp

> Everything the player learns or is told: information portions and what they unlock, the news feed, the encyclopedia, conversations, and the personal-data-assistant contact list.

**Needs** — [`Actor.h`](Actor.h.md) · [`InfoPortion.h`](InfoPortion.h.md) · [`encyclopedia_article.h`](encyclopedia_article.h.md) · [`GametaskManager.h`](GametaskManager.h.md) · [`GameTaskDefs.h`](GameTaskDefs.h.md) · [`PhraseDialog.h`](PhraseDialog.h.md) · [`character_info.h`](../xrServerEntities/character_info.h.md) · [`relation_registry.h`](relation_registry.h.md) · [`map_manager.h`](map_manager.h.md) · [`Level.h`](Level.h.md) · [`Inventory.h`](Inventory.h.md) · [`CustomDetector.h`](CustomDetector.h.md) · [`alife_registry_wrappers.h`](alife_registry_wrappers.h.md) · [`game_object_space.h`](game_object_space.h.md) · [`UIGameSP.h`](UIGameSP.h.md) · [`ui/UITalkWnd.h`](ui/UITalkWnd.h.md) · [`ui/UIMainIngameWnd.h`](ui/UIMainIngameWnd.h.md) · [`ai/trader/ai_trader.h`](ai/trader/ai_trader.h.md) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: registry bookkeeping and list maintenance; no device or format concern

## Purpose

The player's *knowledge* is a first-class simulated quantity in this game, and this file is
where it is changed. An **information portion** is the unit: an authored record, named by
string, that the player either has or does not have. Receiving one is how the world tells
the player something, and a portion is a bundle of consequences — encyclopedia entries to
add and to retract, tasks to give, conversations to unlock.

Everything else in the file follows from that: the news feed that reports changes, the
conversation surface whose available topics depend on what the player knows, and the
contact list that tracks who the player has met.

It is a separate file from the rest of the player only by size; it is the player's own
implementation, split for readability.

## State

None owned locally. The player's knowledge lives in **alife registries** — persistent,
save-backed collections keyed to the player that survive level changes, because what the
player knows is not a property of the level they are standing in. Three are used here:

```text
REGISTRY known_info      : list<InfoRecord>       # which portions the player has, with when
REGISTRY encyclopedia    : list<ArticleRecord>    # article id, time received, article kind
REGISTRY game_news       : list<NewsItem>         # the news feed, oldest first
```

The one collection the player owns directly:

```text
RECORD DeferredNews
  news : NewsItem
  due  : int            # global clock reading at which it should be delivered
  # the list is kept sorted by due time DESCENDING, so the soonest item is last
```

Invariant: the deferred list is sorted in reverse so that delivery pops from the end. That
is the whole reason for the inverted ordering — removal from the end of a contiguous list
is free and removal from the front is not.

## `OnReceiveInfo`

**Contract** — the player has just been given an information portion. Delegates first to
the inventory-owner base, which is what actually records the portion and which may refuse
(the player already has it); on refusal nothing else happens. Otherwise loads the portion's
authored record and applies its consequences: encyclopedia changes, then tasks, then a
script callback naming the portion, then a refresh of any open conversation.

**Invariants** — the base's answer is a *gate*. Every consequence in this file is applied
at most once per portion, and that is enforced by the base's duplicate check, not here.

```text
FUNCTION on_receive_info(info_id) -> bool
  IF NOT base.on_receive_info(info_id) THEN RETURN false     # already known
  portion = load_authored_portion(info_id)
  apply_encyclopedia_changes(portion)
  give_tasks(portion)
  fire_script_callback(inventory_info, self, info_id)
  IF a conversation screen is open THEN mark its topic list stale
  RETURN true
```

**Notes** — the return value is *not* "did I accept the portion": the function returns
false both when the portion was already known and when there is no single-player UI at
all. Callers use it as the former. A rebuild should separate the two.

## `AddEncyclopediaArticle`

**Contract** — applies one portion's encyclopedia changes to the persistent article list:
first removes every article the portion retracts, then appends every article it adds that
is not already present. Each addition is stamped with the current game time and its kind,
fires a script callback carrying the article's group, name and kind, and refreshes the
assistant screen.

**Invariants** — **retractions are applied before additions**, so a portion may
simultaneously withdraw an article and add its replacement. Reversing the order would
delete the replacement.

```text
FUNCTION apply_encyclopedia_changes(portion)
  FOR EACH id IN portion.articles_to_remove
    remove every article with that id from the registry
  FOR EACH id IN portion.articles_to_add
    IF the registry already holds that id THEN CONTINUE     # idempotent
    article = load_authored_article(id)
    registry.append(id, current_game_time, article.kind)
    fire_script_callback(article_info, self, article.group, article.name, article.kind)
    refresh the assistant screen
```

**Notes**

- The removal pass gathers all retractions before compacting the list once, rather than
  compacting per retraction. That is a cost decision on a list that can hold hundreds of
  articles and is walked on every portion.
- Each article's full authored record is loaded just to read its kind and its display
  names, and then discarded. Only the identifier, the timestamp and the kind are stored;
  the text is re-loaded when the screen shows it. A rebuild should keep that split — the
  article bodies are localized text and belong in the string table, not in the save.
- The routing of an article to a specific assistant section by kind exists in the original
  as dead code for one of the three games. This fork refreshes the whole screen instead.

## `AddGameTask`

**Contract** — gives the player every task the portion names, through the task manager.
A portion with no tasks does nothing. A missing portion is tolerated with a log line
rather than a failure, because a portion can be named by a script that has a typo in it
and losing the whole frame to that is worse than losing one task.

## `AddGameNews` / `AddGameNews_deffered` / `UpdateDefferedMessages` / `ClearGameNews`

**Contract** — the news feed. Immediate delivery stamps the item with the current game
time, shows it on the heads-up display if there is one, and appends it to the persistent
feed. Deferred delivery instead queues the item against a wall-clock deadline. The update
pass delivers every item whose deadline has passed. Clearing empties both the feed and the
pending queue.

**Invariants**

- The receive timestamp is assigned at *delivery*, not at queueing, so a deferred item
  reads as having arrived when the player saw it.
- The deferred queue is kept sorted with the soonest item last, and delivery walks from
  the end and stops at the first item not yet due. The whole pass is therefore proportional
  to the number of items actually delivered, not to the queue length.
- Queueing re-sorts the entire queue. The queue holds a handful of items in practice, so
  this is not a problem, but it is a sort where an ordered insertion would do.

```text
FUNCTION deliver_due_news()
  WHILE queue is not empty
    item = queue.last                 # the soonest
    IF item.due > now THEN BREAK
    deliver(item.news)
    queue.remove_last
```

**Notes** — deferred delivery exists so that a burst of consequences from one event
(finishing a task gives three news items) arrives spaced out rather than stacked. The
delay is on the engine's wall clock, not the game clock, so it does not scale with the
game's time compression and does not survive a save — a pending item is lost. That is
accepted because the items are cosmetic.

## `OnDisableInfo`

**Contract** — the player is losing an information portion. Notes whether they actually
had it, delegates to the base, and fires a removal callback into script *only if they
did* — so a script never sees a removal that did not happen. Refreshes any open
conversation.

**Notes** — losing a portion does not undo its consequences: articles stay, tasks stay.
Retraction is expressed by a portion that removes articles, not by removing the portion
that added them. That asymmetry is the authored model and a rebuild must preserve it.

## `UpdateAvailableDialogs`

**Contract** — recomputes which conversation topics the player may raise with a given
partner. Clears the current lists, then adds every topic named by **every information
portion the player holds**, then adds every topic the partner's own character record
offers to a player, then lets the conversation base filter the result by each topic's own
preconditions.

**Invariants** — this walks the player's entire knowledge set and loads each portion's
authored record, every time a conversation's topic list needs refreshing. That is why the
refresh is triggered by explicit staleness marks after a knowledge change rather than
being recomputed per frame.

```text
FUNCTION update_available_dialogs(partner)
  clear available and already-checked topic lists
  FOR EACH info_record IN player.known_info
    portion = load_authored_portion(info_record.id)
    FOR EACH topic IN portion.dialog_names
      offer_topic(topic, partner)
  FOR EACH topic IN partner.character_info.topics_offered_to_the_player
    offer_topic(topic, partner)
  base.update_available_dialogs(partner)      # applies each topic's own preconditions
```

**Notes** — the two sources are asymmetric on purpose. Topics from the player's knowledge
are things the player has learned to ask about; topics from the partner's character record
are things that character will discuss with anyone.

## `TryToTalk` / `RunTalkDialog` / `StartTalk`

**Contract** — beginning a conversation. The player offers to talk to whoever they are
currently looking at, and only if not already talking. The partner may refuse — the offer
is a question, not a command — and on acceptance the conversation begins, any open screen
is dismissed, and the talk screen opens with the partner's own "this conversation cannot
be broken off" flag applied.

**Invariants** — starting a conversation **hides an active detector**, because the player
holds it in one hand and the talk screen shows the character's hands empty. This is the
kind of cross-cutting rule that a rebuild will miss and only notice as a visual glitch.

## `NewPdaContact` / `LostPdaContact`

**Contract** — a character has entered or left the player's assistant contact list.
Gaining a contact animates the contact indicator — silently for a monster, because a
monster is a contact but not a person to be notified about — and puts a marker on the map
coloured by the relationship. Losing one removes every relationship-coloured marker for
that entity and the dead-body marker as well.

**Invariants**

- Gaining a contact is single-player only; in a networked game the contact list is not
  maintained.
- Removal iterates *every* relationship kind rather than the one the marker was added
  under, because the relationship may have changed since. The marker name is derived from
  the relationship, so the only safe removal is to try them all. A rebuild that stores the
  marker's identity at insertion does not need this.

## `ReceivePhrase`

**Contract** — the conversation partner has said something. Marks the open talk screen's
topic list stale before letting the conversation base record the phrase, so the screen
re-reads after the phrase's own effects have applied.

## `OnDialogSoundHandlerStart` / `OnDialogSoundHandlerStop`

**Contract** — asks whether a conversation partner voices its lines. Only traders do;
anyone else is silent and the pair returns "not handled", which lets the caller fall back
to text only. Starting passes the phrase so the right voice line is chosen.

**Notes** — restricting voiced dialogue to one character class is data-like knowledge
living in a type check. A rebuild should ask the partner whether it has a voice.
