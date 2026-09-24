# src/xrGame/profile_data_types_script.h

> Declares the profile registrator and the callback type an asynchronous profile operation reports through.

**Needs** — [`profile_data_types.h`](profile_data_types.h.md) · [`mixed_delegate.h`](mixed_delegate.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`profile_data_types_script.cpp`](profile_data_types_script.cpp.md) · [`profile_store.h`](profile_store.h.md)
**Tier floor** — T3: a type alias and a registration declaration

## Purpose

Declares two things used across the profile subsystem:

- **`store_operation_cb`** — the completion callback shape every asynchronous profile
  operation takes: *(succeeded, message)*. It is a **mixed delegate**, meaning one value
  can hold either a native function or a Lua function, because the caller may be either
  side. The message on failure is a string-table key, not display text.
- **the registration hook** that exports the award and best-score record shapes to the
  script layer; implemented in
  [`profile_data_types_script.cpp`](profile_data_types_script.cpp.md).

**Notes** — the original carries a standing complaint here that this one type alias is
expensive to compile, because naming it drags the whole binding layer's template machinery
into every file that includes it. That is a C++ build-time problem with no behavioural
content; a rebuild in a language with a real module system deletes the concern along with
the file.
