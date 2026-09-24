# src/xrGame/ai/monsters/control_combase.h

> The contract every monster control element satisfies: a lifecycle, an optional resource half, an optional client half, and the four shapes those two halves combine into.

**Needs** — [`control_com_defs.h`](control_com_defs.h.md) · [`control_manager.h`](control_manager.h.md) · [`basemonster/base_monster.h`](basemonster/base_monster.h.md)
**Used by** — [`anim_triple.cpp`](anim_triple.cpp.md) · [`anim_triple.h`](anim_triple.h.md) · [`anti_aim_ability.h`](anti_aim_ability.h.md) · [`burer_fast_gravi.cpp`](burer/burer_fast_gravi.cpp.md) · [`burer_fast_gravi.h`](burer/burer_fast_gravi.h.md) · [`control_animation.h`](control_animation.h.md) · [`control_animation_base.h`](control_animation_base.h.md) · [`control_critical_wound.h`](control_critical_wound.h.md) · [`control_direction.h`](control_direction.h.md) · [`control_direction_base.h`](control_direction_base.h.md) · [`control_jump.h`](control_jump.h.md) · [`control_manager.cpp`](control_manager.cpp.md) · [`control_manager_custom.h`](control_manager_custom.h.md) · [`control_melee_jump.h`](control_melee_jump.h.md) · _and 9 more_
**Tier floor** — T2: an interface hierarchy with virtual dispatch on a per-frame path

## Purpose

An interface header, and therefore the substance holder for the whole control-bus design.
Chapter 24's creatures are not written as one class per creature with one update function;
they are assembled from *control elements* that seize each other. This file says what an
element is.

The key idea is that an element has **two independent halves**, and which halves it has
decides what it can do:

- the **controlled** half makes it a *resource*: it owns a payload record, it can be
  captured by exactly one other element at a time, and it can be locked out of the update
  list while still being captured;
- the **controlling** half makes it a *client*: it can capture resources, it can subscribe
  to bus events, and it is told when its capture starts and stops.

An element with only the controlled half is a **pure** element — the four body resources.
An element with only the controlling half is a **base** element — the default driver of a
pure element, the one that is reinstated whenever nobody else wants the resource. An
element with both is a **custom** element — a creature ability, which both seizes the body
and can itself be seized by a higher-layer com. That three-way split is what the manager's
ordering rules key off, and it is inferred from which halves are present rather than
declared, which is the one thing in this design a rebuild should improve.

## State

```text
RECORD ControlElement                 # the lifecycle half, present on every element
  manager   : ControlManager          # the bus this element is attached to
  creature  : Monster                 # the creature this element drives
  active    : bool                    # is this element in the manager's per-frame list
  inited    : bool                    # has reinit run at least once

RECORD ControlledHalf                 # present on resources
  capturer  : optional<ControlElement>  # who currently drives this resource
  locked    : bool                      # captured, but excluded from the update list

RECORD ControllingHalf                # present on clients
  controlled : list<ControlElement>   # declared but unused; see Notes
```

**Invariants** — a resource has at most one capturer. `active` and `locked` are separate
facts and both must be checked before an element is updated: an ability that has seized
the path channel and then drives the path itself *locks* it, so the path's own base
driver stops running while the seizure stands.

## `CControl_Com` — the lifecycle

**Contract** — the half every element has. `init_external` binds the element to a manager
and a creature; `load` reads the creature's configuration section; `reinit` resets to a
known state at spawn and at every respawn; `reload` re-reads configuration after a section
change. `update_schedule` runs on the creature's scheduled (rate-degraded) tick and
`update_frame` on every rendered frame — an element implements whichever it needs and the
manager calls both.

`set_active` flips the flag *and* calls the activate or deactivate hook, so an element
never has to be told twice.

`check_start_conditions` answers "may I begin now" and defaults to yes. Abilities override
it to no by default (see `CControl_ComCustom`), which makes "can this creature do this
right now" a question that must be answered explicitly rather than assumed.

**Notes** — the manager and creature pointers are set once and never cleared; the element
does not own either. `reinit` sets `active` false and `inited` true, and the combining
shapes below then re-activate the element if it is one that should run by default.

## `CControl_ComControlled` — the resource half

**Contract** — what a resource must offer. `data` returns the payload record that its
current capturer writes into; `on_capture` is called when a new capturer takes it and
resets the payload to defaults; `on_release` when the capture ends. `reset_data` is the
one operation an implementor usually overrides.

**Invariants** — the payload is *reset on capture, not on release*. The capturer therefore
starts from defaults and the previous capturer's settings never leak forward, but a
resource sitting uncaptured still holds the last values written — which is what lets a
base driver take over mid-motion without a visible snap.

## `CControl_ComControlling` — the client half

**Contract** — what a client must offer. `on_start_control` and `on_stop_control` bracket a
capture, named by channel, and are where a client subscribes to and unsubscribes from the
events it cares about. `on_event` receives a bus event with its payload.

**Notes** — the list of controlled elements this half declares is never filled anywhere in
the codebase; a client tracks its own captures. It should not be reproduced.

## The four combining shapes

These exist because C++ needed them named; what matters is the four *roles*, which a
rebuild must still distinguish because the manager treats them differently.

| Shape | Halves | Reinit leaves it | Role |
|---|---|---|---|
| **pure** | controlled + payload | active | a body resource: movement, path, direction, animation |
| **base** | controlling | active | the default driver of one resource; captures it during its own reinit |
| **custom** | both + payload | *inactive*, start conditions default to **no** | a creature ability |
| **storage** | controlled + payload | — | the payload-owning mixin the other three build on |

**Notes** — the asymmetry in the last column is the design. Pure and base elements come up
running, because a creature must always be able to move, face and animate. A custom
element comes up dormant and refuses to start until something asks it to, because an
ability that started itself would fight the state machine.

The ordering in which the three are reinitialized is not arbitrary and is enforced by the
manager, not here: pure first (they must exist to be captured), base second (they capture
during reinit), custom last.
