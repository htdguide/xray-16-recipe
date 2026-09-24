# src/xrGame/aimers_base_inline.h

> Samples what a set of bones *would* look like under a candidate animation, on a scratch animation channel, without disturbing the pose the player is currently seeing.

**Needs** — [`aimers_base.h`](aimers_base.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md)
**Used by** — [`aimers_base.cpp`](aimers_base.cpp.md) · [`aimers_base.h`](aimers_base.h.md)
**Tier floor** — T2: it drives the animation system's blend machinery directly.

## Purpose

An aimer must answer a hypothetical: *if* this character played that animation, where would
the muzzle point? Answering it needs the animation evaluated — but evaluating it on the
live blend set would visibly change the pose. This file is the trick that makes the
evaluation invisible.

## `fill_bones`

**Contract** — Given a list of bone indices of interest, fills in each one's pose under the
sampled motion, in both the skeleton's local space and world space. Temporarily suspends
the root bone's own hook, plays the motion on a dedicated scratch channel, reads the bones,
closes the channel, and restores the hook. Leaves the visible animation state as it found
it. Touches global animation state, so it is not safe to run concurrently with the frame's
own animation update.

```text
FUNCTION fill_bones(bones_of_interest) -> local poses, world poses
  remember the root bone's hook and its parameter
  IF we are not sampling a starting animation, or the object is not mid-blend
    clear the root bone's hook       # root motion must not displace the sample
  channel = a scratch animation channel, distinct from the visible one

  IF sampling the start of an animation
    # Reproduce the *current* blend set on the scratch channel, so the sample sees
    # exactly the pose the character is in right now, mid-blend and all.
    FOR EACH bone group
      FOR EACH active blend in that group
        start the same motion on the scratch channel and copy the blend's whole state
  ELSE
    # Sample the candidate motion at its final frame.
    FOR EACH bone group
      start the candidate motion on the scratch channel
      set its time to just under its total length

  FOR EACH bone of interest
    read its animated pose using only the scratch channel
    world pose = reference transform composed with the local pose

  close the scratch channel on every bone group
  restore the root bone's hook
```

**Invariants** — Three things must be undone before returning, and they are undone in the
reverse order of doing them: the scratch cycles are closed on every bone group, and the
root hook is restored last. Leaving either behind corrupts the visible character — a live
scratch channel double-weights the pose, and a lost root hook detaches root motion.

The final-frame time is set to the motion's length *minus one sample interval and an
epsilon*, not to the length itself. Sampling exactly at the end either wraps to the first
frame of a looping motion or falls off the end of a one-shot one; backing off by a single
sample lands on the last real pose.

**Notes** — The scratch channel is the whole design. The animation system supports several
independent channels whose results are combined; by using one nobody renders from, the
aimer gets a full animation evaluation for free without a separate offline pose evaluator.
A rebuild whose animation system has no channels needs either a scratch skeleton instance
or a pose-evaluation entry point that does not touch the live blend set.

The requirement that the bone-index array be no longer than the identifier array is checked
at compile time, since both are fixed-size and supplied by the derived aimer.
