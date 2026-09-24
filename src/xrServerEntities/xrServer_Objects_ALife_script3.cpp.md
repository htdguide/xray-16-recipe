# src/xrServerEntities/xrServer_Objects_ALife_script3.cpp

> Exports the hanging lamp, the physics object with a yaw setter, and the smart terrain.

**Needs** — [`xrServer_Objects_ALife_Monsters.h`](xrServer_Objects_ALife_Monsters.h.md) · [`xrServer_script_macroses.h`](xrServer_script_macroses.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2.

## Purpose

The third part of the alife export. Two ordinary registrations and one that matters a great
deal.

## `cse_alife_object_hanging_lamp`

**Contract** — a light fixture: a dynamic visual object with a ragdoll, so it can be shot
down and swing.

## `cse_alife_object_physic`

**Contract** — a rigid-body prop, plus **`set_yaw`**: write the record's orientation about
the vertical axis.

**Notes** — this is a single-axis alias for the orientation field that
[`xrServer_Objects_script.cpp`](xrServer_Objects_script.cpp.md) already exports whole. It
exists because rotating a prop about the vertical is what scripts actually do, and reading
the whole orientation, modifying one component and writing it back is three script
operations rather than one. Added by this project rather than the original; a rebuild may
skip it without affecting shipped scripts.

## `cse_alife_smart_zone`

**Contract** — the **smart terrain**: a restrictor volume that is also scheduled, exported at
the **zone** level. That level is what makes this registration significant — see
[`xrServer_script_macroses.h`](xrServer_script_macroses.h.md). A script subclass may override
every decision a smart terrain makes: the per-tick update, a creature entering, whether the
terrain is enabled for a given creature, how suitable it is for one, registering and
unregistering a creature, handing that creature a job, and the detection probability.

**Notes** — every smart terrain in the shipped games is a script class subclassing this. It
is the single largest thing in the game that is implemented in script rather than in the
engine, and this one registration is what makes that possible. A rebuild that drops the
script layer must reimplement the entire job-assignment system natively.
