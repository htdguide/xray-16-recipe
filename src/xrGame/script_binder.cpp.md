# src/xrGame/script_binder.cpp

> Attaches a script-authored behaviour to a game object from its configuration, forwards the object's whole lifecycle to it, and detaches it rather than propagating any failure.

**Needs** — [`script_binder.h`](script_binder.h.md) · [`script_binder_object.h`](script_binder_object.h.md) · [`script_game_object.h`](script_game_object.h.md) · [`GameObject.h`](GameObject.h.md) · [`xrServer_Objects_ALife.h`](../xrServerEntities/xrServer_Objects_ALife.h.md) · [`ai_space.h`](ai_space.h.md) · [`Level.h`](Level.h.md) · [`xrScriptEngine/script_engine.hpp`](../xrScriptEngine/script_engine.hpp.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — reached through its declarations in [`script_binder.h`](script_binder.h.md); callers name that, not this file.
**Tier floor** — T2: an optional dispatch through the script virtual machine, on every entity lifecycle event

## Purpose

This is how a modder makes an object *do* something without an engine change. An object's
configuration section may name a script function; when the object loads, that function is
called with the object's script facade and may attach a binder to it; from then on every
lifecycle event the engine raises on the object is forwarded to that binder.

The file makes two decisions and they are the whole of it: **when the attachment happens**,
and **what a failing script costs.**

## State

See [`script_binder.h`](script_binder.h.md).

## The failure policy

**Contract** — every forwarded call is guarded, and every guard does the same thing: **catch
the failure, detach and destroy the binder, and continue.** The engine-side object survives;
its scripted behaviour does not.

```text
FUNCTION forward(hook, args...)
  IF nothing attached THEN RETURN <the neutral answer>
  TRY
    RETURN attached.hook(args...)
  ON any failure
    clear()                 # destroy the binder; this object is engine-only from now on
    RETURN <the neutral answer>
```

**Invariants**

- **A script error never takes the object with it, and never takes the process with it.**
  This is the policy the modding ecosystem rests on: a broken binder on one crate does not
  end the session. It is the same reasoning as the evaluator adapter's answer-false rule
  (see
  [`script_property_evaluator_wrapper.cpp`](script_property_evaluator_wrapper.cpp.md)),
  applied to a whole behaviour rather than to one question.
- Detaching is **permanent for that object's life**. There is no retry and no diagnostic at
  the detach site — the script engine has already reported whatever raised. So a binder that
  fails once silently stops existing, and the visible symptom is an object that quietly stops
  behaving. A rebuild should log the detach; it is the single most confusing failure mode in
  the script layer.
- The destroy path itself is guarded: if destroying the binder fails, the reference is
  dropped anyway. That leaks rather than hangs, which is the right trade at teardown.

## `reload(section)`

**Contract** — the attachment point, and the only place a binder is ever created. Runs when
the object loads its configuration. Requires nothing to be attached yet.

```text
FUNCTION reload(section)
  REQUIRE nothing attached
  IF section has no "script_binding" key THEN RETURN      # most objects have none

  handler = script_engine.lookup_function(section.script_binding)
  IF handler NOT FOUND THEN
    log script error "function <name> is not loaded!"
    RETURN

  TRY
    handler(owner.script_facade)      # the handler decides whether to attach, and what
  ON any failure
    clear(); RETURN

  IF something was attached THEN
    TRY attached.reload(section)      # let the binder read its own configuration
    ON any failure DO clear()
```

**Invariants**

- The handler is **given the object and attaches a binder as a side effect** — it does not
  return one. That inversion is what lets one handler decide, from the object it is handed,
  which of several binder classes to attach, or none at all. A rebuild wanting a return value
  instead must accept that the handler can no longer decline based on the object.
- The binding is named in *configuration*, keyed per section, so which objects are scripted is
  data rather than code. A section with no such key is simply not scripted, which is the
  common case and must be cheap: the key lookup is the only cost for the vast majority of
  objects.
- The binder's own configuration read happens *after* attachment, so the binder can already
  see its object when it reads its section.

**Notes** — the whole attachment path can be compiled out by a debug switch, which disables
scripting for every object at once. It is a bisection tool, not a feature.

## `set_object`

**Contract** — attaches a binder to this object. **Refuses in multiplayer**: outside single
player the binder is destroyed immediately instead of attached, silently.

**Invariants** — this is a policy statement, not a guard. Per-object script behaviour is a
single-player feature, because in multiplayer the authoritative state lives on a server that
is not running these scripts. A rebuild that wants scripted objects in multiplayer has to
decide where the script runs before it can lift this.

Attaching twice is a contract violation, asserted: one object, one binder, for its life.

## `net_Spawn`

**Contract** — forwarded when the object is spawned from its server record, and **it is the
one forwarded hook whose answer matters**: the binder may refuse the spawn, and the refusal
propagates to the engine, which abandons the object. A binder that is absent, or whose
object's server record is not an alife record, answers *accept*.

**Invariants** — the neutral answer here is **accept**, not refuse. An object with no script
must always spawn. A rebuild that defaults to refuse will spawn nothing.

## `net_Destroy`

**Contract** — forwards the destroy hook and then releases the binder unconditionally, guard
or no guard. The end of the attachment's life, and the only unconditional release outside
`clear`.

## `shedule_Update(elapsed)`

**Contract** — forwarded once per scheduled update with the elapsed time. This is the
binder's per-frame entry point and the hot path of the whole script layer, since it runs for
every scripted object at whatever rate the scheduler grants it.

## `save` / `load`

**Contract** — forwarded so a binder can persist its own state into the object's save record
and read it back. The binder's bytes sit inside the object's, so a binder that saves and a
binder that does not are not interchangeable across a save.

**Invariants** — if the save call fails and the binder is detached mid-write, the object's
save record is left with a partial script payload and the matching load will read garbage.
Nothing detects that. A rebuild should write the script payload length-prefixed so a failed
or absent binder is skippable on load.

## `net_SaveRelevant`

**Contract** — asks the binder whether this object is worth saving at all. **The neutral
answer is refuse**: an object with no binder answers *not relevant* from this mixin, and its
engine-side relevance is decided elsewhere. Note this is the opposite polarity from the spawn
hook, and for a different reason — here the mixin is contributing an opinion, not gating.

## `net_Relcase(object)`

**Contract** — forwarded when some *other* object is about to be destroyed, so the binder can
drop any reference it holds to it. The other object is translated to its script facade before
being handed over. Non-game objects are ignored.

**Invariants** — this is the script layer's half of the engine's "no destroyed entity is
referenced" invariant (see the system requirements' runtime invariants). A binder holding a
stale object reference across a destruction is exactly the bug this hook exists to prevent,
and a rebuild that omits it will find script-held references outliving their objects.

## `init` / `clear`

**Contract** — `init` marks nothing attached; `clear` destroys the attached binder, tolerating
a failure in the destruction, and then marks nothing attached. Construction runs `init`;
destruction asserts nothing remains attached, which means every object must have gone through
either `net_Destroy` or `clear` first.
