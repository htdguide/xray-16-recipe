# src/xrGame/ai/trader/trader_animation.cpp

> Implements the trader's animation: three callbacks into the script layer, fired whenever a motion or a spoken phrase runs out.

**Needs** — [`trader_animation.h`](trader_animation.h.md) · [`ai_trader.h`](ai_trader.h.md) · [Seam: Audio device](../../../../SYSTEM-REQUIREMENTS.md#seam-audio-device) · [Seam: Script virtual machine](../../../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — [`trader_animation.h`](trader_animation.h.md)
**Tier floor** — T3: three completion tests and three callbacks

## Purpose

The inversion this file implements is the point: **the engine asks and the script answers.**
Everywhere else in the game, the script tells the engine what to play. Here the engine
notices that something has finished and raises a callback whose job is to supply the next
thing. That gives dialogue authors frame-accurate control over a talking head without the
engine knowing anything about dialogue.

Three callbacks exist and they fire on three different exhaustion conditions:

| Callback | Fires when |
|---|---|
| spoken phrase ended | the sound stopped playing |
| body animation requested | the body motion handle is clear |
| head animation requested | the head motion handle is clear *and* a phrase is still playing |

The third is the one that produces lip-and-gesture behaviour: head motions are re-requested
only while the trader is actually saying something, so a silent trader's head holds still
while his body keeps idling.

## `update_frame`

**Contract** — called every frame by the trader, but only when neither a script action nor a
script animation is in control. Not re-entrant with respect to the callbacks it raises: a
callback that sets a new animation writes the handle this function is about to read, which
is why each branch reads its handle before raising.

```text
FUNCTION update_frame()
  IF a sound exists
    IF the sound is still playing
      move it to the head bone's world position      # the voice comes from the mouth
    ELSE
      raise script callback: spoken phrase ended
      destroy the sound

  IF body_motion IS ABSENT
    raise script callback: body animation requested
    # if the callback supplied one, invalidate the head motion too, so that the
    # head is re-requested against the new body animation rather than continuing
    # a gesture that belonged to the previous one
    IF body_name IS SET
      head_motion = absent

  IF head_motion IS ABSENT
    IF a sound exists AND it is still playing
      raise script callback: head animation requested
```

**Invariants**

- The head is re-requested *after* the body in the same frame, so a script that answers the
  body request immediately gets its head request in the same frame rather than the next. The
  ordering is load-bearing for the gesture to line up with the phrase.
- The head-invalidation test reads the *name* of the last body animation, not the handle.
  Since the name is only ever set and never cleared, the test is true from the first body
  animation onward. A rebuild can simplify it to "always", and should note that the
  condition as written would have meant something different had the name ever been cleared.

## `set_animation` / `set_head_animation`

**Contract** — resolve a motion by name from the trader's model and play it, registering the
corresponding completion callback. Both store the name. Neither checks that the motion
resolved, so a name the model does not carry produces an invalid handle and, in a release
build, undefined play behaviour. The names come from dialogue data, so a rebuild should
validate them at load.

## `set_sound`

**Contract** — plays a sound *and* a head animation together, as one operation. Destroys any
sound already playing first. Creates the sound as a world-positioned effect and starts it at
the trader.

**Notes** — a disabled alternative would have played the phrase as a non-positional
two-dimensional sound, which is how a first-person conversation is usually mixed. The
shipped trader's voice is positional and attenuates with distance, which is why walking away
mid-sentence fades the trader out.

## `external_sound_start` / `external_sound_stop`

**Contract** — the dialogue system's entry point. Start destroys any current sound, creates
and plays the new phrase, and clears the head motion so that the update loop immediately
asks the script for a matching gesture. Stop destroys the sound.

**Invariants** — the difference between this path and the paired one above is which side
chooses the head animation: the paired form is told which one to play, the external form
leaves it to the next frame's callback. Dialogue uses the external form, so every spoken
line's gesture is chosen per frame by script rather than fixed at the start of the line.

## `sound_position`

**Contract** — composes the trader's world transform with the head bone's current local
transform and returns the resulting origin. Recomputed every frame while a phrase plays, so
the voice tracks the head through the animation rather than sitting at the trader's origin.

## `remove_sound`

**Contract** — stops the sound if it is playing, destroys it, and releases the handle.
Requires a sound to exist; every caller checks first.

## Notes

**The external-sound flag is dead.** It is cleared at re-initialisation and read nowhere.
Presumably it once distinguished the two sound paths so that the completion callback could
be suppressed for dialogue-driven phrases; today both paths raise the same callback.
