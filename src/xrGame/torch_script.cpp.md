# src/xrGame/torch_script.cpp

> Exports the torch, the personal data assistant and the four detector variants to the
> script layer, as constructible classes and nothing more.

**Needs** — [`Torch.h`](Torch.h.md) · [`PDA.h`](PDA.h.md) · [`SimpleDetector.h`](SimpleDetector.h.md) · [`EliteDetector.h`](EliteDetector.h.md) · [`AdvancedDetector.h`](AdvancedDetector.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a binding table.

## Purpose

Six classes are registered with the script engine, each under its own name, each derived
from the script-visible game object, each with a no-argument constructor and **no methods at
all**.

A registration with no methods is not pointless, and what it buys is worth stating once for
every `_script.cpp` sibling in this chapter. It does two things. It makes the class **name**
exist, so a script can spawn one and so the binder can attach a script object to an instance
of it. And it establishes the **derivation**, so that an instance handed to a script as a
game object can be narrowed to this class and back. Everything a script actually does with a
torch it does through the game object facade.

That six unrelated item classes are registered from the file named for one of them is an
arbitrary grouping. A rebuild should register each class beside its own definition; nothing
depends on the grouping.

## State

Stateless.

## `script_register(engine)`

**Contract** — registers `CTorch`, `CPda`, `CScientificDetector`, `CEliteDetector`,
`CAdvancedDetector` and `CSimpleDetector` as script classes deriving from the game object,
each constructible with no arguments. Runs once at script-engine startup. The names are
addressed by shipped scripts and are frozen.

**Notes** — the registered name and the class it stands for differ in one case: the
scientific detector's class name in the source is not the one the scripts use. A rebuild must
carry the *registered* names across exactly, not the internal ones.
