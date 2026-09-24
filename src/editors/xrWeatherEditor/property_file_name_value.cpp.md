# src/editors/xrWeatherEditor/property_file_name_value.cpp

> A text row that names a file, carrying the five settings its chooser needs.

**Needs** — [`property_file_name_value.hpp`](property_file_name_value.hpp.md) · [`property_string.hpp`](property_string.hpp.md) · [`property_file_name_value_base.hpp`](property_file_name_value_base.hpp.md)
**Used by** — reached through its declarations in [`property_file_name_value.hpp`](property_file_name_value.hpp.md); callers name that, not this file.
**Tier floor** — T2: a managed refinement of the accessor-bound text adapter

## Purpose

Reading and writing are unchanged from [`property_string.cpp`](property_string.cpp.md). All this adds is the five chooser settings, converted from native text once at registration and held for the row's life.

## State

```text
RECORD FileNameProperty EXTENDS TextProperty
  default_extension : text
  filter            : text
  initial_directory : text     # the stored value is relative to this
  title             : text
  remove_extension  : bool
```

## `construct(getter, setter, extension, filter, directory, title, remove_extension)`

**Contract** — Copies the five settings, already converted to presentation text by the registration site, and the value callbacks to the base. Allocates.

## `default_extension` · `filter` · `initial_directory` · `title` · `remove_extension`

**Contract** — Each returns the correspondingly named stored field. Contracts in [`property_file_name_value_base.hpp`](property_file_name_value_base.hpp.md).

**Notes** — The settings are fixed at registration and cannot change. That is a genuine limitation rather than an oversight: the directory a weather texture is chosen from depends on where the game data lives, which the engine resolves once at startup. A rebuild whose asset roots can change while the editor runs has to make these queries, like the live label lists.
