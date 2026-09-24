# src/xrGame/stalker_animation_head.cpp

> The head channel: three-way choice between a talking head, a listening head and a still one, driven by whether this stalker is the one currently speaking.

**Needs** — [`stalker_animation_manager.h`](stalker_animation_manager.h.md) · [`stalker_animation_data.h`](stalker_animation_data.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`sound_player.h`](sound_player.h.md) · [`UIGameSP.h`](UIGameSP.h.md) · [`ui/UITalkWnd.h`](ui/UITalkWnd.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a three-branch selection

## Purpose

The smallest of the channels. The head has exactly two motions — still and moving — and the
whole file is the question of when the mouth should be moving.

## `assign_head_animation`

**Contract** — return the head motion for this frame. Never returns nothing; the head always
has something to play.

```text
FUNCTION assign_head_animation() -> motion
  # 1. This stalker is the player's conversation partner and a line is playing:
  #    move the head regardless of what the sound system thinks.
  IF the talk screen is open
     AND its partner is this stalker
     AND it is playing a line
    RETURN the moving head motion

  # 2. Otherwise the sound player decides. No audible sound: still.
  IF the stalker has no active sound  RETURN the still head motion

  # 3. A sound that is NOT one of the wordless danger vocalizations: moving.
  IF the active sound is not a wordless danger sound  RETURN the moving head motion

  # 4. A wordless danger vocalization: still.
  RETURN the still head motion
```

**Invariants** — the talk-screen branch has to come first and has to bypass the sound test,
because dialogue lines are played by the conversation system rather than by the stalker's
own sound player, so the sound player does not know about them. Without this branch a
stalker talking to the player would stand there with its mouth shut.

The last two branches together say: move the head for anything with words in it, keep it
still for the grunts and shouts. Those are authored as the danger sound mask, and the
distinction is audible — a stalker shouting a warning should not appear to be having a
conversation.

**Notes** — the two motions are the whole head vocabulary, indexed by literal position in the
head table. A rebuild should name them.

## `head_play_callback`

**Contract** — the renderer's notification that the head motion ended. Tells the channel and
nothing more; the head has no deferred callback, because nothing outside cares when a head
motion finishes.
