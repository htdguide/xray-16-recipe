# src/xrGame/PdaMsg.h

> Two small records: one log entry for a message exchanged through the personal data assistant, and one record of having talked to somebody.

**Needs** — [`alife_space.h`](../xrServerEntities/alife_space.h.md) · [`pda_space.h`](pda_space.h.md)
**Used by** — [`PDA.h`](PDA.h.md) · [`alife_registry_container_composition.h`](alife_registry_container_composition.h.md)
**Tier floor** — T3: plain records

## Purpose

The device keeps a history — what was said, to whom, when — and the dialogue system keeps a
parallel record of who has been spoken to. Both are small enough that they live in one header,
and neither has an implementation file: they are shapes, and the code that fills them is
elsewhere.

## State

```text
RECORD PdaMessage                  # one entry in the message log
  msg      : PdaMessageKind        # which message; the kinds live in the device's own space
  received : bool                  # true = incoming, false = sent by us
  question : bool                  # true = a question, false = an answer
  info_id  : text                  # the knowledge item this message concerns
  time     : int                   # the in-game time it was sent or received

RECORD TalkContact                 # one record of having spoken to somebody
  id   : entity id                 # the character; initialized to "nobody"
  time : int                       # the in-game time of the contact
```

**Invariants**

- The two booleans are orthogonal and both are needed: a message is one of four things, and the
  device's log renders each differently. Collapsing them into one direction flag loses the
  question/answer distinction that drives which replies are offered.
- `info_id` names a *knowledge item* — the game's unit of "the player now knows this" — rather
  than carrying text. All displayed text comes from the localization table, so a save is
  language-independent.
- The contact time is *in-game* time, not wall time. Contact records are compared against the
  world clock to decide what counts as recent, and the world clock runs far faster than real
  time.

**Notes** — `TalkContact` has a default state meaning "no contact", which is how an empty slot
is expressed without a separate presence flag. A rebuild with an optional type should use it.

Both records are stored in saved games as part of the character's own state, so their field set
is frozen against existing saves but not against anything outside the engine.
