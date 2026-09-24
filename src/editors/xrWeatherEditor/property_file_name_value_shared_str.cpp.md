# src/editors/xrWeatherEditor/property_file_name_value_shared_str.cpp

> The file-naming row, bound to an interned-text slot.

**Needs** — [`property_file_name_value_shared_str.hpp`](property_file_name_value_shared_str.hpp.md) · [`property_string_shared_str.hpp`](property_string_shared_str.hpp.md) · [`property_file_name_value_base.hpp`](property_file_name_value_base.hpp.md)
**Used by** — reached through its declarations in [`property_file_name_value_shared_str.hpp`](property_file_name_value_shared_str.hpp.md); callers name that, not this file.
**Tier floor** — T2: a managed refinement over an engine-owned interned-text handle

## Purpose

The interned-text twin of [`property_file_name_value.cpp`](property_file_name_value.cpp.md), and the one that carries the weather document's texture references — those are interned names the engine resolves against the texture set.

## State

```text
RECORD FileNameInternedProperty EXTENDS InternedTextProperty
  default_extension : text
  filter            : text
  initial_directory : text
  title             : text
  remove_extension  : bool
```

## `construct(engine, slot, extension, filter, directory, title, remove_extension)` · the five settings

**Contract** — Identical to the accessor-bound twin; reading and writing are the facade-routed ones from [`property_string_shared_str.cpp`](property_string_shared_str.cpp.md).

**Notes** — The two file-name adapters duplicate all five fields because the binding flavours are separate bases. This is the clearest case in the directory for making the chooser settings a record a row *has* rather than a base a row *is*: the settings are pure data and have nothing to do with how the value is bound.
