# src/xrGame/PHShellCreator.h

> Declares the default physics-shell builder, implemented in [`PHShellCreator.cpp`](PHShellCreator.cpp.md).

**Needs** — [`ph_shell_interface.h`](ph_shell_interface.h.md)
**Used by** — [`PHShellCreator.cpp`](PHShellCreator.cpp.md) · [`Weapon.h`](Weapon.h.md) · [`physic_item.h`](physic_item.h.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `CPHShellSimpleCreator`, one filling of the shell-builder interface. Substance is in
[`PHShellCreator.cpp`](PHShellCreator.cpp.md).

What the header decides is that building a body is a **strategy the object chooses**, not
something the object does. The interface has one method and objects mix in whichever
implementation suits them: this one derives everything from the model's skeleton, while a
character or a vehicle supplies its own. A rebuild should keep the seam — it is the only
thing separating "an object with a body" from "an object that knows how bodies are made".

Exported units:

- `CPHShellSimpleCreator` — build a shell from the object's skeleton, if it has a visual and
  does not already have one.
