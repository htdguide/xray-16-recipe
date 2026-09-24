# src/utils/mp_balancer/pch.cpp

> Exists so the aggregation header has a compilation unit; it contains no code.

**Needs** — [`pch.h`](pch.h.md)

**Used by** — reached through its declarations in [`pch.h`](pch.h.md); callers name that, not this file.

**Tier floor** — T4: it is a build artifact.

## Purpose

A build-system fixture with no runtime meaning. It survives a rebuild as nothing.

## State

Stateless.
