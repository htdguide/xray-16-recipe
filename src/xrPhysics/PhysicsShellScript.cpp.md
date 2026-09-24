# src/xrPhysics/PhysicsShellScript.cpp

> The slice of the physics object model that Lua may touch, and the names it touches it by.

**Needs** — [`PhysicsShell.h`](PhysicsShell.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: pure declaration of an exported surface; nothing here computes.

## Purpose

Three classes are exported to the script layer — a shell, an element and a joint — and this
file is the complete list of what a mod author can do to physics. That list is a *contract*,
not an implementation detail:
[conformance criterion 10](../../SYSTEM-REQUIREMENTS.md#6-conformance) requires every shipped
script to run unmodified, which fixes these names, their argument order and their overload
resolution exactly.

Read it as a boundary definition. What is *absent* is as load-bearing as what is present: no
script can create, destroy, build or activate a shell, register it for collision, or change
its mass. Scripts may push things, read things, and unlock or lock breaking — nothing that
could leave the world in a state the engine did not construct.

## Stateless.

## `physics_shell`

**Contract** — exported as `physics_shell`, with:

| script name | meaning |
|---|---|
| `apply_force` | push the whole shell, by three components |
| `get_element_by_bone_name`, `get_element_by_bone_id`, `get_element_by_order` | the three lookups; `order` is store order, which is dense and stable |
| `get_elements_number` | how many elements |
| `get_joint_by_bone_name`, `get_joint_by_bone_id`, `get_joint_by_order` | the same three for joints |
| `get_joints_number` | how many joints |
| `block_breaking`, `unblock_breaking`, `is_breaking_blocked` | suspend and resume fracture |
| `is_breakable` | whether it can fracture at all |
| `get_linear_vel`, `get_angular_vel` | velocity read-back |

**Notes** — both a bone-id and a store-order lookup are exported, and they are not the same
number. Bone ids come from the model; store order is the shell's own list index. Scripts that
iterate use order (it is dense, `0` to `count-1`); scripts that target a named body part use
the bone name. A rebuild that unifies the two breaks shipped scripts.

## `physics_element`

**Contract** — exported as `physics_element`, with `apply_force`, `is_breakable`,
`get_linear_vel`, `get_angular_vel`, `get_mass`, `get_density`, `get_volume`, `fix`,
`release_fixed`, `is_fixed`, and `global_transform` — the last returning the element's live
world transform by value.

**Notes** — `fix` and `release_fixed` are the only *mutating* structural operations exposed
anywhere in this file, and they are safe precisely because pinning and unpinning an existing
element cannot leave the world holding a reference to something that does not exist. That is
the rule a rebuild should use when deciding what else may be exported.

`global_transform` is synthesised at the binding layer rather than being a method on the
element, because the underlying read-back writes through an out-parameter and script callers
want a value. That mismatch — out-parameters on one side, return values on the other — is the
most common shape of adapter in this whole binding surface.

## `physics_joint`

**Contract** — exported as `physics_joint`, with `get_bone_id`, `get_first_element`,
`get_stcond_element` (the shipped spelling of *second*), `get_axes_number`, `is_breakable`;
the anchor setters in all three coordinate systems (`set_anchor_global`,
`set_anchor_vs_first_element`, `set_anchor_vs_second_element`); the axis-direction setters in
the same three (`set_axis_dir_global`, `set_axis_dir_vs_first_element`,
`set_axis_dir_vs_second_element`); `set_limits`; the motor pair
`set_max_force_and_velocity` / `get_max_force_and_velocity`; and the readouts `get_axis_angle`,
`get_limits`, `get_axis_dir`, `get_anchor`.

**Invariants** — `get_max_force_and_velocity` and `get_limits` each return *two* values to the
script rather than writing through arguments. Shipped scripts destructure both, so a rebuild
must return a pair from these two specifically, even if its own convention is otherwise.

**Notes** — the misspelling `get_stcond_element` is frozen. It is in the shipped scripts; a
corrected name is a new name and breaks them. If a rebuild wants the spelling fixed, it must
export both and keep the old one forever.

The joint surface is the widest of the three because this is how mods build and tune
machinery — doors, lifts, mounted guns — at run time. Note what is still missing even here:
spring and damping factors are settable but breakability thresholds are not, so a script can
make a joint stiff but cannot make an unbreakable thing break.
