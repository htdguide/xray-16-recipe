# src/xrGame/CustomZone.h

> Declares the anomaly base class implemented in [`CustomZone.cpp`](CustomZone.cpp.md), its five states, its twenty configured flags and the per-resident record.

**Needs** — [`space_restrictor.h`](space_restrictor.h.md) · [`xrEngine/Feel_Touch.h`](../xrEngine/Feel_Touch.h.md)
**Used by** — [`Actor_Feel.cpp`](Actor_Feel.cpp.md) · [`AmebaZone.cpp`](AmebaZone.cpp.md) · [`AmebaZone.h`](AmebaZone.h.md) · [`CustomDetector.h`](CustomDetector.h.md) · [`CustomMonster.cpp`](CustomMonster.cpp.md) · [`CustomZone.cpp`](CustomZone.cpp.md) · [`GraviZone.cpp`](GraviZone.cpp.md) · [`GraviZone.h`](GraviZone.h.md) · [`HairsZone.cpp`](HairsZone.cpp.md) · [`HairsZone.h`](HairsZone.h.md) · [`Helicopter2.cpp`](Helicopter2.cpp.md) · [`Mincer.cpp`](Mincer.cpp.md) · [`MosquitoBald.cpp`](MosquitoBald.cpp.md) · [`MosquitoBald.h`](MosquitoBald.h.md) · _and 15 more_
**Tier floor** — T3: a declaration plus one threshold constant

## Purpose

Declares the anomaly as a space restrictor that also feels touch — a volume the pathfinder
avoids, which notices what is inside it. Substance in
[`CustomZone.cpp`](CustomZone.cpp.md).

One constant lives here and is used throughout: the **small-object radius**, 0.6 metres.
Every size-dependent decision in an anomaly — which entrance effect, which hit effect,
which give-up timeout, whether the zone ignores you at all — is this one comparison.

## The five states

```text
idle        # nothing active inside; duration is unbounded by definition
awaking     # something entered; the wind-up
blowout     # the discharge: effects on a schedule, then damage, then maybe an artefact
accumulate  # recharge; returns to blowout if something is still inside, else to idle
disabled    # switched off; no residents, no effects
```

## The configured flags

Twenty booleans, all read from the anomaly's configuration section. Grouped by what they
decide:

```text
# who the zone cares about
ignore_nonalive, ignore_small, ignore_artefact
# presentation
blowout_wind, blowout_light, idle_light, idle_light_volumetric, idle_light_shadow,
idle_light_r1, bolt_entrance_particles, idle_object_particles_dont_stop,
affect_pick_dof
# behaviour
use_on_off_time, use_secondary_hit, spawn_blowout_artefacts, visible_by_detector
# runtime, not configured
zone_is_active, blowout_wind_active, fast_mode, always_fastmode
```

## The per-resident record

```text
RECORD SZoneObjectInfo
  object          : game object
  small_object    : bool      # frozen at entry
  nonalive_object : bool      # frozen at entry; re-checked live by the give-up rules
  zone_ignore     : bool      # the zone has stopped caring
  particles       : list      # effects attached to this object by this zone
  time_in_zone    : int       # milliseconds
  time_affected   : real      # stamped at entry; unread by the base
```

## Exported units

- `CCustomZone` — the anomaly.
- `Load`, `net_Spawn`, `net_Destroy`, `net_Import`, `net_Export`, `save`, `load` — the
  lifecycle. Persistence keeps one byte: disabled, or idle.
- `UpdateCL`, `shedule_Update`, `UpdateWorkload` — the fast tick, the slow tick, and the
  state advance both of them call.
- `IdleState`, `AwakingState`, `BlowoutState`, `AccumulateState` — the four state
  handlers, each answering whether it changed state.
- `SwitchZoneState`, `OnStateSwitch` — request a state change over the network; apply one
  when it arrives.
- `Enable`, `Disable`, `ZoneEnable`, `ZoneDisable`, `IsEnabled`, `ZoneState` — the
  on/off surface.
- `UpdateOnOffState`, `GoEnabledState`, `GoDisabledState` — the authored duty cycle.
- `feel_touch_new`, `feel_touch_delete`, `feel_touch_contact`, `feel_touch_on_contact` —
  admission, entry classification and exit.
- `enter_Zone`, `exit_Zone` — the per-resident hooks a subclass extends.
- `Affect` — **the one abstract operation**: what this anomaly does to one resident. The
  base does nothing.
- `AffectObjects` — run it over every resident, once per frame at most.
- `RelativePower`, `Power`, `effective_radius`, `CalcDistanceTo` — the falloff, and the
  nearest-sub-shape distance it is measured against.
- `CreateHit` — build and emit a hit attributed to the zone, or to whoever deployed it.
- `Hit` — being shot: a small effect, no damage.
- `UpdateBlowout` — the blowout's five-effect schedule.
- `BornArtefact`, `ThrowOutArtefact`, `PrefetchArtefacts`, `SpawnArtefact` — the
  pre-spawned artefact pool and its release.
- `StartWind`, `UpdateWind`, `StopWind` — driving the global weather wind during a
  blowout.
- `StartIdleLight`, `UpdateIdleLight`, `StopIdleLight`, `StartBlowoutLight`,
  `UpdateBlowoutLight`, `StopBlowoutLight` — the two lights.
- `PlayIdleParticles`, `StopIdleParticles`, `PlayAccumParticles`, `PlayAwakingParticles`,
  `PlayBlowoutParticles`, `PlayEntranceParticles`, `PlayHitParticles`,
  `PlayBulletParticles`, `PlayBoltEntranceParticles`, `PlayObjectIdleParticles`,
  `StopObjectIdleParticles` — the presentation surface.
- `OnMove` — feed the anomaly's own velocity to its effects when it moves.
- `OnEvent` — state change, and the two artefact ownership transfers.
- `net_Relcase` — drop every reference to an object being destroyed.
- `o_switch_2_fast`, `o_switch_2_slow`, `AlwaysTheCrow`, `light_in_slow_mode` — the
  distance-driven update-rate switch and whether the idle light survives slow mode.
- `ef_anomaly_type`, `ef_weapon_type` — the two numbers by which the AI evaluation layer
  classifies an anomaly and the threat it poses.
- `GetMaxPower`, `SetMaxPower`, `GetHitType` — the tuning surface scripts and subclasses
  reach.
