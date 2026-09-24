# src/xrGame/Missile.h

> Declares the thrown-item base class implemented in [`Missile.cpp`](Missile.cpp.md).

**Needs** — [`hud_item_object.h`](hud_item_object.h.md) · [`HudSound.h`](HudSound.h.md) · [`Missile.cpp`](Missile.cpp.md)
**Used by** — [`ActorAnimation.cpp`](ActorAnimation.cpp.md) · [`Actor_Weapon.cpp`](Actor_Weapon.cpp.md) · [`Bolt.cpp`](Bolt.cpp.md) · [`Bolt.h`](Bolt.h.md) · [`Grenade.cpp`](Grenade.cpp.md) · [`Grenade.h`](Grenade.h.md) · [`Missile.cpp`](Missile.cpp.md) · [`agent_explosive_manager.cpp`](agent_explosive_manager.cpp.md) · [`game_cl_capturetheartefact_buywnd.cpp`](game_cl_capturetheartefact_buywnd.cpp.md) · [`game_sv_deathmatch.cpp`](game_sv_deathmatch.cpp.md) · [`object_handler_planner.cpp`](object_handler_planner.cpp.md) · [`object_handler_planner_missile.cpp`](object_handler_planner_missile.cpp.md) · [`object_property_evaluators.cpp`](object_property_evaluators.cpp.md) · [`stalker_animation_manager_impl.h`](stalker_animation_manager_impl.h.md) · _and 2 more_
**Tier floor** — T3: a declaration only

## Purpose

Declares `CMissile`, the base of everything a character throws — grenades above all. Substance
is in [`Missile.cpp`](Missile.cpp.md), including the fake-missile mechanism that is the class's
whole reason for existing.

Exported units:

- `CMissile` — the thrown item.
- `EMissileStates` — four states appended to the held-item state set: the wind-up, the charged
  hold, the release and the follow-through. They are numbered from one past the last base
  state, so the base's state numbering is part of this class's contract.
- `Load` / `reinit` / `net_Spawn` / `net_Destroy` / `Destroy` — the lifecycle.
- `UpdateCL` / `shedule_Update` — the per-frame charge and the low-rate fuse check.
- `Action` — the two throw gestures, fixed-force and charged.
- `State` / `OnStateSwitch` / `OnAnimationEnd` / `OnMotionMark` — the animation binding. The
  launch is driven from the motion marker.
- `Throw` / `setup_throw_params` / `spawn_fake_missile` — the launch itself.
- `OnEvent` — the two ownership events that attach and launch the fake missile.
- `OnH_A_Chield` / `OnH_B_Independent` / `OnActiveItem` / `OnHiddenItem` — the ownership and
  selection transitions.
- `activate_physic_shell` / `setup_physic_shell` / `create_physic_shell` / `PH_A_CrPr` — the
  physics body, in its thrown and its lying-there forms.
- `ExitContactCallback` — the contact filter that stops a thrown item colliding with its
  thrower. Static, because the physics layer calls it without an instance.
- `net_Relcase` — clears that filter's reference when the thrower goes away.
- `UpdateXForm` / `UpdatePosition` / `UpdateFireDependencies_internal` — placing the item in the
  hands and deriving the launch direction.
- `render_item_ui` / `render_item_ui_query` — the charge meter.
- `throw_point_offset` / `destroy_time` / `set_destroy_time` / `time_from_begin_throw` /
  `ef_weapon_type` / `GetBriefInfo` / `AlwaysTheCrow` / `cast_missile` — accessors and the
  down-cast hook the rest of the item hierarchy uses to recognize a thrown item without a type
  test.
