# src/xrGame/ui/UITextVote.cpp

> A vote dialog that let a player type an arbitrary vote subject. Entirely commented out; kept as a
> record of the removed feature.

**Needs** — [`UITextVote.h`](UITextVote.h.md)
**Used by** — [`UITextVote.h`](UITextVote.h.md)
**Tier floor** — T4: the file contributes nothing to the build

## Purpose

The multiplayer voting menu (see [`UIVotingCategory`](UIVotingCategory.cpp.md)) offers seven fixed
subjects. This screen was the eighth route: an edit box into which a player typed a free-text vote
subject, which was then issued as a console command.

**Every line of it is commented out.** It still appears in the module's source list, so it is
compiled and produces nothing. Nothing in the repository references the type. It is recorded because
a rebuild reading the voting menu will notice the eighth branch and wonder what it was.

## State

`Stateless.` No code is compiled.

## The removed screen

For the record, what it did:

- built five widgets from the voting layout document under the element `text_vote` — a background, a
  header, an edit box, and confirm and cancel buttons;
- on confirm, if the edit box was non-empty, issued a vote-start console command whose argument was
  the typed text, prefixed with a marker distinguishing a free-text subject from a fixed one, then
  told the multiplayer game state the menu was finished;
- on cancel, told the game state the menu was finished without a vote.

**Notes** — The feature is unreachable in two independent ways: nothing constructs the type, and the
voting menu's eighth case is empty. Taking a player-typed string and pasting it into a console
command line is also the kind of thing a rebuild should not reproduce literally — the vote subject
should reach the server as data, not as a command to be parsed.

The whole path ran through a matchmaking-era service described at
[Seam: Multiplayer matchmaking and accounts](../../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts).
