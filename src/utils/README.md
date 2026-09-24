# src/utils — the tooling

Chapter 28 of [the build order](../../SYSTEM-REQUIREMENTS.md#7-build-order). Four programs
that are not the game, one format declaration left behind by a fifth, one dead loader, and
— by an accident of directory layout — the maths library the whole engine is built on.

## What this chapter is responsible for

Everything here runs *beside* the engine rather than inside it: it builds the data the
engine reads, checks the data a client claims to be running, derives the balance numbers a
designer then edits by hand, or serves a player's record to a website. None of it is
linked into the game, none of it is on any frame budget, and none of it is required for the
engine to run.

That does not make it optional to read. One of these programs **writes a frozen format**
that the engine only knows how to read, and it is the sole specification of that format's
producer. Another is the **server half of a two-sided protocol** whose client half is in
the game. A third leaves behind the only record of **how the multiplayer balance set was
derived**. A fourth is a service that **reaches nothing today** and is here for the shape
of its data rather than for its code.

## Where it sits

At the end of the build order, resting on the core layer and on nothing above it, with one
exception that is not really part of this chapter: `xrMiscMath` is chapter 3, a foundation
everything else depends on, and it is physically in this directory for historical reasons
only. It is covered by [its own chapter](xrMiscMath/README.md); ignore its presence here.

The four programs share the core layer — virtual filesystem, configuration parser,
checksum, compressor — with one deliberate exception: the profile server shares nothing
with the engine at all, not even that, and vendors its own dependencies.

## Load-bearing ideas, named once

**A tool that writes a format the engine reads is part of the format's specification.**
The engine's chapters describe reading. The packer describes writing, and the two must
agree byte for byte or no shipped data loads. Where a decision looks arbitrary on the
reading side — why compression is signalled by a size equality rather than a flag, why
offsets are absolute, why volumes are capped — the reason is on the writing side.

**Tools resolve their inputs through the game's own filesystem description, not through
paths.** The balance tool and the configuration verifier both mount the game's logical
roots exactly as the engine does, which is why they must be run from inside a real
installation. The packer is the deliberate exception: it mounts exactly one folder as the
entire namespace, with no archives and no parent roots, and that is what makes an
archive's entry names come out relative to the folder the operator named.

**Command lines here are recovered by substring search over the raw line, not parsed.**
Three of the four programs do it. The consequence is that an option's value must be
separated from its name by exactly one space, and that a path containing another option's
text is misread. Preserve the option *names* — build scripts use them — not the scanning.

**Two things in this chapter are dead and say so.** The profile server's account service
was shut down in 2014
([Seam: Multiplayer matchmaking and accounts](../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)),
and the pixel loader is referenced by no build description at all. Both are recipe'd for
the decisions they record, and both pages say plainly that the code reaches nothing.

**Vendored third-party source is excluded from the recipe.** The profile server checks in
three complete outside source trees to make its build reproducible. They are somebody
else's reasoning, and describing them would describe the wrong system — see
[`mp_gpprof_server/libraries/README.md`](mp_gpprof_server/libraries/README.md).

## The sub-chapters

| Directory | Role |
|---|---|
| [`xrCompress`](xrCompress/README.md) | **The archive packer** — the write side of the frozen archive format, plus the patch-differencing pass |
| [`mp_balancer`](mp_balancer/README.md) | **The balance authoring tool** — flattens the previous game's configuration into an editable multiplayer set, with a designer answering the disagreements |
| [`mp_configs_verifyer`](mp_configs_verifyer/README.md) | **The configuration anti-cheat** — verifies a client's signed configuration dump against the server's own data |
| [`mp_gpprof_server`](mp_gpprof_server/README.md) | **The profile server** — serves a player's multiplayer record over the web from a service that no longer answers |
| [`xrLC_Light`](xrLC_Light/README.md) | What is left of the lighting compiler: the baked-light record the renderer still reads |
| [`xrMiscMath`](xrMiscMath/README.md) | Not part of this chapter — vectors, matrices and quaternions, [chapter 3](../../SYSTEM-REQUIREMENTS.md#7-build-order) |

## The loose files

| File | Role |
|---|---|
| [`xrLoadSurface.cpp`](xrLoadSurface.cpp.md) | Decodes a texture to a flat pixel block for a tool that needs to look at it; built by nothing today |

One further file in the directory is build description — it names the maths library as the
only sub-project built as part of the engine — and carries no decisions.
