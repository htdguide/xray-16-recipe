# src/xrSound/guids.cpp

> Build scaffolding: one translation unit that emits the reverb extension's interface identifiers.

**Needs** — [`stdafx.h`](stdafx.h.md) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: it exists to control where constants are emitted.

## Purpose

The vendor reverb extension identifies its property sets by globally unique identifiers declared in
its headers. Those identifiers must be *defined* in exactly one translation unit and merely
*declared* everywhere else, so this file is that unit — it includes the header with the definition
switch set, and contains nothing else. It is deliberately excluded from the precompiled header, since
the switch must not leak into any other file.

**Notes** — This is a pure ecosystem artefact. A rebuild reaching the same extension will have its
own answer for where a constant lives; a rebuild on any other reverb will delete this file. The
identifiers themselves are the extension's, not the engine's, and are not reproduced here.
