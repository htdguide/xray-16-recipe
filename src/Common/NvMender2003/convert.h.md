# src/Common/NvMender2003/convert.h

> Copies a three-component vector between the engine's own vector type and the tangent generator's.

**Needs** — [`NVMeshMender.h`](NVMeshMender.h.md)
**Used by** — [`mender_input_output.h`](mender_input_output.h.md)
**Tier floor** — T4: field-for-field assignment between two identical shapes.

## Purpose

The vendored tangent generator was written against a different library's vector type than
the engine's. Both are three floats in the same order, so the conversion is a copy — but it
has to be written somewhere, and this file is the somewhere.

This file exists only because the generator was adopted rather than rewritten. A rebuild
that writes its own tangent generator against its own vector type deletes it.

## State

Stateless.

## Vector conversion

**Contract** — copy the three components in both directions, returning the destination so calls
can be chained. No conversion, no validation, no allocation.
