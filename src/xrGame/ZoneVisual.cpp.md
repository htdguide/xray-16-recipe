# src/xrGame/ZoneVisual.cpp

> An anomaly whose blowout is an animation: it idles on one motion and plays an attack motion at authored offsets within the blowout timeline.

**Needs** — [`ZoneVisual.h`](ZoneVisual.h.md) · [`CustomZone.h`](CustomZone.h.md) · [`xrServer_Objects_ALife_Monsters.h`](../xrServerEntities/xrServer_Objects_ALife_Monsters.h.md) · [`Include/xrRender/KinematicsAnimated.h`](../Include/xrRender/KinematicsAnimated.h.md) · [`Include/xrRender/RenderVisual.h`](../Include/xrRender/RenderVisual.h.md)
**Used by** — reached through its declarations in [`ZoneVisual.h`](ZoneVisual.h.md); callers name that, not this file.
**Tier floor** — T2: motion lookup and timeline comparison

## Purpose

Most anomalies are invisible volumes that announce themselves through particles. This one
has a *model* — a skinned mesh that idles and lunges — so its attack has to be driven by
the skeleton rather than by an emitter. The file's whole job is to bind two named motions
to the zone's blowout state machine and to fire them at the right points in the blowout's
own clock.

The two motion names come from the entity's server record, not from its configuration
section, because different placed instances of the same anomaly class use different
models and therefore different motion names. The two *timing* values come from the
section, because they are a property of the class's behaviour.

## State

```text
RECORD VisualZoneState
  idle_motion    : motion handle    # played on spawn and after each attack
  attack_motion  : motion handle    # played once per blowout
  attack_start   : int              # milliseconds into the blowout state at which the attack motion begins
  attack_end     : int              # milliseconds at which the zone returns to idle
```

Invariant: `attack_start < attack_end`, checked at load. The two are compared against the
blowout state's own elapsed time, so they are offsets within the blowout, not absolute
times.

## `net_Spawn`

**Contract** — completes the client object from its server record. Resolves both motion
names against the loaded model's animation bank and fails hard, naming the object, the
motion and the model, if either is missing — an anomaly with an unresolvable attack motion
would silently never animate, which is far harder to diagnose than a spawn-time failure.
Starts the idle motion and makes the object visible.

**Invariants** — the base spawn must succeed first; on its failure nothing here runs. The
object is explicitly made visible here rather than inheriting the zone default, because
the zone base hides itself (an ordinary anomaly has nothing to draw).

```text
FUNCTION net_spawn(server_record)
  IF NOT base.net_spawn(server_record) THEN RETURN failure

  record = server_record as visual-zone record
  skeleton = visual as animated skeleton
  attack_motion = skeleton.find_motion(record.attack_animation)
  REQUIRE attack_motion valid   # message names object, motion and model
  idle_motion   = skeleton.find_motion(record.startup_animation)
  REQUIRE idle_motion valid

  skeleton.play(idle_motion)
  set_visible(true)
  RETURN success
```

## `Load`

**Contract** — reads the two blowout-relative timings from the configuration section after
the base has read its own keys. Both are required; there is no default, because an
animated anomaly with no attack window would never attack.

## `SwitchZoneState`

**Contract** — the zone state-machine transition hook. On any departure *from* the blowout
state it restarts the idle motion, then delegates. This is the safety net for blowouts cut
short: the attack motion is started from the timeline and stopped from the timeline, but a
blowout interrupted before its end would otherwise leave the model frozen mid-lunge.

## `UpdateBlowout`

**Contract** — runs per update while the zone is blowing out. Compares the authored attack
window against the interval covered by this update — the previous state time and the
current one — and starts the attack motion when the window's start falls inside it, the
idle motion when the window's end does.

**Invariants** — the test is an interval containment, not an equality, precisely because
the zone is updated at the scheduler's degraded rate: a distant anomaly may advance
hundreds of milliseconds in one step and would step straight over an exact-match test.
Each edge therefore fires at most once per blowout, on the first update whose interval
contains it.

```text
FUNCTION update_blowout()
  base.update_blowout()
  IF attack_start IN [previous_state_time, state_time) THEN skeleton.play(attack_motion)
  IF attack_end   IN [previous_state_time, state_time) THEN skeleton.play(idle_motion)
```

**Notes** — if an update step is long enough to contain both edges, both fire in order and
the attack motion is replaced by the idle one in the same frame — the lunge is skipped
rather than played at the wrong time. That is the intended degradation.
