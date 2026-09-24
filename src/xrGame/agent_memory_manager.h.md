# src/xrGame/agent_memory_manager.h

> Declares the squad's shared perception manager and the three list types it propagates masks over.

**Needs** — [`agent_memory_manager.cpp`](agent_memory_manager.cpp.md) · [`memory_space.h`](memory_space.h.md) · [`agent_memory_manager_inline.h`](agent_memory_manager_inline.h.md)
**Used by** — [`agent_enemy_manager.cpp`](agent_enemy_manager.cpp.md) · [`agent_manager.cpp`](agent_manager.cpp.md) · [`agent_member_manager.cpp`](agent_member_manager.cpp.md) · [`agent_memory_manager.cpp`](agent_memory_manager.cpp.md) · [`agent_memory_manager_inline.h`](agent_memory_manager_inline.h.md) · [`group_hierarchy_holder.cpp`](group_hierarchy_holder.cpp.md)
**Tier floor** — T3: a declaration.

## Purpose

Declares the surface implemented in
[`agent_memory_manager.cpp`](agent_memory_manager.cpp.md). Names the three perception
record kinds the squad shares — seen, heard, hit-by — and the list type of each; the record
definitions themselves belong to the perception layer.

Exported units:

- **`update`**, **`remove_links`** — the squad manager's uniform driving pair.
- **`set_squad_objects`** — three overloads installing the seen, heard and hit-by lists the
  squad shares; the manager does not own them.
- **`visibles`**, **`sounds`**, **`hits`** — access to those installed lists.
- **`reset_memory_masks`** — spread knowledge to every combat member.
- **`update_memory_masks`**, **`update_memory_mask`** — delete a departing member's bit
  from every mask.
- **`object_information`** — latest squad-wide sighting of one object.

**Notes** — The file's own header comment names it as the inline file and the inline file
names it as the header; the two are transposed and neither name matters.
