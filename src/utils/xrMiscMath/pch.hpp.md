# src/utils/xrMiscMath/pch.hpp

> The one thing this module says about itself: it compiles the *definitions* of symbols
> every other module sees as imported.

**Needs** — [`Common/Common.hpp`](../../Common/Common.hpp.md) · [`xrCommon/math_funcs_inline.h`](../../xrCommon/math_funcs_inline.h.md) · [`xrCore/_std_extensions.h`](../../xrCore/_std_extensions.h.md)
**Used by** — [`pch.cpp`](pch.cpp.md)
**Tier floor** — None: a build artifact with no runtime behaviour

## Purpose

A shared prologue for the module's translation units. Most of it is the mechanical
business of getting the platform prologue and the elementary numeric helpers in front of
every file, which does not survive into a rebuild at all.

One line does survive as a decision. The math types this module implements are *declared*
in the core module's headers, and those headers normally mark the core module's symbols as
coming from elsewhere. Here that marking is flipped, because this is the translation unit
that supplies the bodies. The problem being solved is stated in full in
[the directory README](README.md#why-this-is-a-separate-module): the math bodies must end
up compiled into every module that uses them rather than reached across a shared-library
boundary, while still being written once. Any rebuild with a module system that can link
one implementation into many consumers deletes this file and the flip with it.

The source carries a maintainer's note that the module should be dissolved. Take it as
read: the split is a build-system workaround, not a design.

## State

`Stateless.`

## Exported units

None. This file declares nothing of its own.
