# src/xrGame/ai/monsters/control_manager.cpp

> The bus itself: it owns one control element per channel, arbitrates capture, routes events, and drives the active set each frame.

**Needs** — [`control_manager.h`](control_manager.h.md) · [`control_combase.h`](control_combase.h.md) · [`control_com_defs.h`](control_com_defs.h.md) · [`control_animation.h`](control_animation.h.md) · [`control_direction.h`](control_direction.h.md) · [`control_movement.h`](control_movement.h.md) · [`control_path_builder.h`](control_path_builder.h.md) · [`basemonster/base_monster.h`](basemonster/base_monster.h.md)
**Used by** — reached through its declarations in [`control_manager.h`](control_manager.h.md); callers name that, not this file.
**Tier floor** — T2: per-frame dispatch over a small map of elements

## Purpose

One instance per creature. It holds the channel-to-element table, decides who may drive
each channel, delivers events between elements, and calls the active elements twice a
frame — once on the frame tick, once on the scheduled tick. Every creature ability in this
chapter is written against this object rather than against the creature.

It is also the *only* place that knows the difference between a pure, a base and a custom
element, and it infers that difference from which halves an element has rather than being
told. That inference is worth making explicit in a rebuild.

## State

```text
RECORD ControlManager
  creature       : Monster
  elements       : map<ControlChannel, ControlElement>   # every element, by channel
  base_elements  : map<ControlChannel, ControlElement>   # the default driver per channel
  active         : list<optional<ControlElement>>        # the per-frame update list
  listeners      : map<ControlEvent, list<ControlElement>>
  animation, direction, movement : ControlElement        # the three it owns outright
  path           : ControlElement                        # borrowed, see Notes
```

**Invariants** — an element that appears in `elements` has had the manager and the creature
bound into it. `active` holds exactly the elements that are both active and unlocked, with
one exception: removal writes `none` into the slot instead of erasing it, and the holes
are compacted only at the end of an update pass. That is what makes it safe for an element
to deactivate itself, or another element, from inside its own update.

`base_elements` is a *subset* of `elements` keyed by the channel the base drives, and its
presence changes what release means: a channel with a registered base always has a driver,
a channel without one goes idle.

**Notes** — the manager constructs and owns the animation, direction and movement elements.
It does **not** own the path builder: the creature constructs that one (it must also be
the creature's movement manager, see [`control_path_builder.cpp`](control_path_builder.cpp.md))
and installs it here. A rebuild that unifies ownership will find nothing depends on the
split except the destructor.

## `capture`

**Contract** — give a channel to a new driver. The previous driver is told its control
stopped and the channel's payload is reset to defaults; the new driver is recorded and
told its control started. Does not activate the channel — capture and activation are
separate steps, so a client can seize a channel, fill in its payload, and only then start
it running.

**Invariants** — the channel must currently be uncaptured or held by a *base* element.
Seizing a channel from another non-base client is an error; a debug build dumps the whole
creature's control state before it asserts, which is the standard way this class of bug is
diagnosed.

```text
FUNCTION capture(client, channel)
  target   <- elements[channel]
  previous <- target.capturer
  REQUIRE previous IS none OR previous IS a base element

  IF target.active THEN
    target.on_release()
    IF previous EXISTS THEN previous.on_stop_control(channel)

  target.capturer <- client
  target.on_capture()          # resets the payload to defaults
  client.on_start_control(channel)
```

## `release`

**Contract** — hand a channel back. If a base driver is registered for the channel, the
channel is immediately re-captured by that base and keeps running. If there is none, the
channel is finalized and deactivated instead. Only the current capturer may release.

```text
FUNCTION release(client, channel)
  target <- elements[channel]
  REQUIRE target.capturer IS client

  IF base_elements HAS channel THEN
    client.on_stop_control(channel)
    target.capturer <- none
    capture(base_elements[channel], channel)     # never goes idle
  ELSE
    IF target.active THEN
      target.on_release()
      deactivate(channel)
    client.on_stop_control(channel)
    target.capturer <- none
```

**Notes** — the two branches differ in ordering, not just in outcome: with a base, the
stop hook runs *before* the payload is reset by the re-capture, so a client cannot write
farewell values into the payload and expect them to survive. Without a base, the release
hook runs first and the client's stop hook last. The asymmetry is almost certainly
accidental and a rebuild should pick one order.

## `capture_pure` and `release_pure`

**Contract** — seize or hand back all four body resources at once, in the fixed order path,
animation, movement, direction. Every ability in this chapter begins with the first and
ends with the second; that is what makes an ability exclusive.

`is_captured_pure` answers whether *any* of the four is held by a non-base client, and is
the standard first line of every ability's start check: abilities do not queue, they
decline.

## `lock` and `unlock`

**Contract** — while holding a pure channel, suspend or resume that channel's own
per-frame update without giving it up. Only the current capturer may lock, and only a pure
channel may be locked.

**Notes** — this exists for one situation and it is worth stating plainly: an ability that
needs the creature to walk a short authored line — the run-up to a jump, the landing
run-out, the skid of a rotation jump — builds that line itself and then locks the path
channel so the path builder will not immediately replan over it. Locking removes the
element from the active list; unlocking puts it back.

## `activate` / `deactivate`

**Contract** — put a channel's element into or out of the per-frame list. Activation also
notifies the creature, which is how a creature reacts to one of its own abilities starting
(the state layer uses this to suppress its normal animation selection). Deactivation marks
the slot empty rather than erasing it, for the reason given under State.

## `notify`, `subscribe`, `unsubscribe`

**Contract** — the event bus. `notify` delivers an event and its payload to every currently
subscribed element, in subscription order; `subscribe` and `unsubscribe` maintain the
per-event list. Unsubscribing swaps the last entry into the removed slot, so subscription
order is **not** stable across an unsubscribe — no receiver may depend on ordering.

**Notes** — delivery is synchronous and re-entrant: handlers routinely capture, release and
notify from inside `on_event`. The listener list is indexed before iteration and is not
copied, so a handler that subscribes to the event it is currently handling can be
delivered to twice or skipped. In practice handlers only ever subscribe to *other* events;
a rebuild should copy the list or defer the mutation.

## `data`

**Contract** — give a client the payload record of a channel, but only if that client is
currently the channel's capturer; otherwise nothing. This single check is what makes the
payload records safe to hand out as raw records: a stale client holding one from a
previous capture gets nothing back and its update silently does nothing, which is the
shape of every ability's update function in this chapter.

## `add`, `set_base_controller`, `install_path_manager`

**Contract** — registration. `add` puts an element on a channel and binds the manager and
creature into it. `set_base_controller` records an element as the fallback driver of a
channel. `install_path_manager` accepts the externally owned path builder. All three run
during creature construction, before any update.

## `path_stop`, `move_stop`, `dir_stop`

**Contract** — the three one-line conveniences every ability opens with: through the
caller's own capture, disable the path, zero the movement target with unbounded
deceleration, and zero the turn rate. They fail loudly if the caller does not hold the
channel.

**Notes** — `move_stop` sets acceleration to the largest representable value rather than
using a flag, so "stop now" and "decelerate at rate *a*" are one code path. That
convention runs through the whole chapter: an unbounded acceleration means *instant*.

## `build_path_line`

**Contract** — on behalf of the current path capturer, replace the creature's path with a
straight line to a point, restricted to a given set of movement speeds. Answers whether a
usable line was produced. The algorithm is in
[`control_path_builder.cpp`](control_path_builder.cpp.md); this is the access check.

## `reinit`

**Contract** — reset every element for a fresh spawn, in three passes: pure elements first,
then base elements, then everything else. Then rebuild the active list from the elements
that came back active and unlocked.

**Invariants** — the three-pass order is required and is not an optimization. A base
element captures its channel during its own reinit, so every pure element must already be
reset. A custom element may consult either during its reset, so it goes last. The file
itself flags the pass structure as something to simplify; the *ordering constraint* is
what a rebuild must keep, not the three loops.

## `com_type`, `get_capturer`, `check_capturer`, `is_captured`

**Contract** — queries. `com_type` finds an element's channel by linear scan, which is fine
at this table size and is used only by diagnostics and by release paths. `is_captured`
reports *non-base* occupancy — a channel driven by its base is considered free, and that
convention is what every ability's start check relies on.

## `check_start_conditions`

**Contract** — ask a channel's element whether it may begin. Pure and base elements always
say yes; abilities say no unless they override, and their overrides are the interesting
part of each ability's twin.

## `update_frame`, `update_schedule`

**Contract** — drive the active list, skipping a dead creature entirely, then compact the
holes left by any deactivation that happened during the pass. Two rates: the frame pass is
every rendered frame, the schedule pass is the engine's rate-degraded tick, which is where
anything that costs a raycast or a path search belongs.

**Notes** — a dead creature's control elements stop updating but are not torn down; the
death animation is driven by the animation channel through a different path, and the
elements must still be intact for the ragdoll and for any ability that needs to clean up.
