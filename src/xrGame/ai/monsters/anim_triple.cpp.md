# src/xrGame/ai/monsters/anim_triple.cpp

> Plays a wind-up, a held or repeated body, and a recovery, in that order, driven by animation completion rather than by a clock.

**Needs** — [`anim_triple.h`](anim_triple.h.md) · [`control_manager.h`](control_manager.h.md) · [`control_combase.h`](control_combase.h.md)
**Used by** — [`anim_triple.h`](anim_triple.h.md)
**Tier floor** — T3: a three-value phase advanced by an event

## Purpose

Almost every creature ability has the same shape: a wind-up the player can read and react
to, a committed middle, and a recovery during which the creature is vulnerable. Expressing
that three times per creature would be twenty copies of the same logic, so it is one
component the abilities configure.

The phases advance on **animation completion**, never on a timer. That is the load-bearing
decision: the wind-up lasts exactly as long as its motion, so re-timing an ability is an
animation change and nothing in code moves. It also means the component must own the
animation channel outright, or another behaviour could interrupt the sequence halfway.

It plugs into the creature's control-component framework
(see [`control_combase.h`](control_combase.h.md)), which is what gives it the capture,
release, event-subscription and start-condition vocabulary.

## State

```text
RECORD AnimationTriple
  config   : TripleAnimationConfig
  phase    : TriplePhase
  previous : TriplePhase       # only needed to tell the first execute tick from the rest
```

**Invariants** — `phase` never moves backward except through the early-exit request, which
jumps it straight to *finalize*. It advances one step per animation completion, except in
*execute*, where it stays put — leaving *execute* is either the looping case (it stays
forever until something else releases the component) or the once case (the second
completion advances it).

## `on_capture`

**Contract** — seizes the animation control channel and subscribes to animation-completion
events. Resets the phase to none. Does not yet seize the movement channels; that happens on
activation.

## `on_release`

**Contract** — releases the animation channel, unsubscribes, and releases whichever of the
path, movement and direction channels the configuration asked for. Releasing exactly what
was captured is what lets an ability freeze a creature in place for its duration without
permanently owning its movement.

## `check_start_conditions`

**Contract** — refuses to start when already running, and refuses when anything else holds
the animation channel. Those are the only two conditions; everything else about whether an
ability is appropriate is the owning behaviour's business.

## `activate`

**Contract** — seizes and immediately halts each configured movement channel, chooses the
starting phase, and plays its motion.

```text
FUNCTION activate()
  FOR EACH channel IN config.capture
    capture(channel) ; halt(channel)      # stop pathing / moving / turning outright

  IF config.skip_prepare THEN
    phase = execute ; previous = prepare  # pretend prepare just ended, so the first
                                          # execute counts as a phase change
  ELSE
    phase = prepare ; previous = none

  advance()
```

**Invariants** — halting each channel as it is captured is what makes an ability stop the
creature dead. A configuration that captures nothing leaves the creature free to keep
walking while the animation plays, which is what a run-attack wants.

**Notes** — the `previous = prepare` assignment in the skip case is not bookkeeping; it is
what makes the phase-change notification fire on the first execute tick. Without it, an
ability that skips its wind-up would never be told its execute phase had begun, and would
never land its hit.

## `advance`

**Contract** — the whole state machine, called on activation and on every animation
completion. Plays the motion for the current phase, raises the phase-change notification at
the right moments, and steps the phase. Does nothing but notify when the sequence is over.

```text
FUNCTION advance()
  IF phase = none THEN
    notify(phase_change, none)     # the sequence is finished; the owner cleans up
    RETURN

  IF phase = execute AND config.execute_once AND previous = execute THEN
    RETURN                         # the single execute has already played out; wait
                                   # for the owner to release, do not replay it

  play the motion for `phase`

  # notify on entering a phase, and on every repeat of a looping execute except
  # the repeats themselves — i.e. exactly once per phase entry
  IF phase != execute OR previous != execute THEN
    notify(phase_change, phase)

  previous = phase
  IF phase != execute THEN phase = the next phase
```

**Invariants** — the phase advances by incrementing through the enumeration, so its order
is the sequence order and its numbering must match the motion array's indexing.

*Execute* is the fixed point: the phase stops advancing there. Leaving it is the owner's
job — either by requesting an early exit, or by releasing the component. That is what
makes a held ability (a creature keeping its shield up) and a one-shot ability (a lunge)
the same component with one flag between them.

**Notes** — the looping case replays the execute motion on each completion and does *not*
re-notify, so a behaviour that must act once per loop cannot use the notification and must
watch the animation itself. Nothing in the shipped creatures needs that.

## `request_finish`

**Contract** — forces the phase to *finalize* and advances immediately, so the recovery
animation plays from wherever the sequence had reached. This is how a held ability is
ended: the owner asks, and the creature plays its recovery rather than snapping out of the
pose.

**Notes** — the original's name for this is an idiom meaning "abort the middle"; the
operation is a graceful stop, not a cancel, and calling it from *prepare* skips the execute
phase entirely, which is a legal and used outcome.

## `play_phase_motion`

**Contract** — writes the phase's motion into the animation channel's shared data block and
marks the block stale, which is what makes the animation layer pick it up on its next pass.
Asserts the channel is held.

**Notes** — the "mark stale" flag rather than a direct play call is the control framework's
convention throughout: components write intent into a shared block and one layer applies it,
so that two components writing in one tick resolve deterministically rather than by call
order.
