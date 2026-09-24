# src/utils/mp_balancer/wpn_collection.cpp

> Rebuilds the multiplayer weapon configuration by flattening the previous game's inherited values into concrete sections and asking a human, once per distinct key, what to do where the patch disagrees.

**Needs** — [`wpn_collection.hpp`](wpn_collection.hpp.md) · [`xr_ini_ex.h`](xr_ini_ex.h.md) · [`pch.h`](pch.h.md) · [`xrCore/LocatorAPI.h`](../../xrCore/LocatorAPI.h.md) · [Data: Configuration](../../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)

**Used by** — nothing in this recipe; entry point or dead code.

**Tier floor** — T3: text configuration, a console conversation, and no timing or layout constraint anywhere.

## Purpose

The three games share a weapon set, and each shipped its own balance for it. When the
newest game's multiplayer balance was rebuilt, the starting point was the previous game's
numbers — but those numbers live behind two levels of indirection that make them unusable
as a starting point: most of a weapon's values are *inherited* from a base section rather
than written on the weapon, and a patch had already overridden an unknown subset of them.

This file collapses both indirections into a set of flat, concrete, human-editable
sections. It does so with a human in the loop, because where the base game and its patch
disagree about a value there is no rule that picks correctly — only a designer who knows
what the patch was for.

It is a one-shot authoring tool. It reads shipped data and produces text a person then
edits by hand; nothing in the engine ever runs it. Its output is configuration, and
configuration is data, so **the tool is not part of a rebuild's critical path at all** —
a rebuilder who has the shipped configuration files does not need to re-derive them. It is
recipe'd because the *rules* it encodes are the only written record of how the
multiplayer balance set was constructed.

## State

```text
RECORD ExtractRule
  prefix : text                 # matches a section whose name starts with it
  keys   : set<text>            # which inherited keys to pull down for such a section

RECORD MergeRun
  previous  : ConfigFile        # the previous game's configuration, read-only
  patch     : ConfigFile        # that game's patch, read-only
  job       : ConfigFile        # the operator's rules
  items     : list<text>        # every multiplayer item, in price-list order
  rules     : list<ExtractRule> # sorted longest-prefix-first
  answered_yes : set<text>      # keys the operator said "always take the patch" for
  answered_no  : set<text>      # keys the operator said "always keep the original" for
  result    : map<text, list<Sect>>   # destination file name -> sections to write
```

**Invariants**

- **The item list comes from the deathmatch price list, not from a weapon list.** The set
  of things that exist in multiplayer is defined by what can be bought, so the price
  section is the authoritative enumeration and anything absent from it is not a
  multiplayer item however much it looks like one.
- **Rules are matched by prefix, longest first.** A rule's prefix is a section-name stem
  such as the family a weapon belongs to, and a more specific rule must beat a more
  general one. The ordering is established once, by sorting the rules by prefix length
  descending, so matching is the first hit in a linear scan.
- **A rule's name is also the destination file name.** There is exactly one output file
  per rule, and every section matching that rule is appended to it. This conflation is
  arbitrary — a rebuild may separate "which keys to extract" from "where the result goes" —
  but it is why the job description is a single line per output file.
- **An operator's standing answer is remembered per key, not per section.** "Always take
  the patch value for this key" applies to that key in every remaining section. That is
  the entire point of the tool: a designer answers a few dozen questions instead of a few
  thousand.
- The two standing-answer sets are disjoint by construction — a key is added to at most
  one, and once added is never asked about again.

## `load_all_mp_weapons`

**Contract** — opens the previous game's configuration, its patch's configuration and the
operator's job description, all read-only and all resolved through logical filesystem
roots rather than literal paths, then reads the extraction rules and enumerates the
multiplayer item set. Fails hard if any of the three is missing. Prints the item set so the
operator can see what the run will cover.

```text
FUNCTION load_all()
  patch    <- open_config(root "$patch_config$" / "system.ltx",  read_only)
  previous <- open_config(root "$game_config$"  / "system.ltx",  read_only)
  job      <- open_config(root "$app_data_root$" / "export_settings.ltx", read_only)
  load_settings()

  FOR EACH item IN section(previous, "deathmatch_base_cost").items
    items.append(item.key)              # the price list *is* the multiplayer item set
    report(item.key)
```

**Notes**

- Both configuration files are the *root* configuration of their game, not a weapons
  file: the tool relies on the format's include mechanism to have pulled in everything by
  the time the parse returns. That is why one path names the whole game's data.
- The two are opened read-only, which is also what enables inheritance to be resolved —
  see [`xr_ini_ex.cpp`](xr_ini_ex.cpp.md). A writable open would silently refuse the
  parent lists this tool exists to consult.

## `load_settings`

**Contract** — reads the job description's rule section, one rule per line: the line's key
is both a section-name prefix and an output file name, and its value is the
comma-separated set of inherited keys to pull down for matching sections. Reserves an
empty output group per rule so the later grouping never has to test for absence. Sorts the
rules longest-prefix-first. Fails hard on a line with no key.

**Notes**

- A rule with no value is legal and means "pull down no inherited keys" — such a section
  is written out with only its own lines.

## `get_extract_keys`

**Contract** — returns the rule governing a section, or nothing when no rule matches. The
match is a prefix comparison over the rule's own length, so a rule named for a family
matches every member of it. First hit wins, and because the rules were sorted
longest-first, first hit is the most specific.

**Notes**

- Sorting by prefix *length* rather than by prefix is a proxy for specificity that is
  correct only when the prefixes nest. Two rules of equal length that both match are
  resolved by whichever the sort happened to leave first, which is not stable. In practice
  the shipped job description has no such pair; a rebuild should resolve ties by taking
  the longest actual common prefix, or reject an ambiguous job description outright.

## `extract_all_params` — the merge

**Contract** — for every multiplayer item, finds its rule, builds its flattened section,
and files the result under the rule's output group. An item with no matching rule is
reported and skipped rather than silently dropped. Interactive: it blocks on operator
input for each unanswered disagreement. Reads both configurations; writes nothing to disk.

```text
FUNCTION extract_all()
  FOR EACH item IN items
    rule <- get_extract_keys(item)
    IF rule IS none
      report_error("no extract rule for " + item)
      CONTINUE
    flattened <- build_section(section(previous, item), rule.keys)
    result[rule.prefix].append(flattened)
```

## `build_section` — what a flattened section contains

**Contract** — produces one output section from one source section. Takes the inherited
keys the rule asked for, in parent order, then the source section's own keys — but only
those the section actually *introduces*. Never fails on a missing key; fails hard on a
parent section that does not exist, which would mean the configuration set is broken.

```text
FUNCTION build_section(source, wanted_inherited_keys) -> Sect
  out.name    <- source.name
  out.parents <- source.parents          # carried so the written header can restate them

  # 1. Pull down the requested keys from each parent, in header order.
  FOR EACH parent IN source.parents
    FAIL WITH missing_parent IF NOT section_exists(previous, parent)
    copy_params(out, section(previous, parent), wanted_inherited_keys)

  # 2. Keep the section's own lines, minus those that only restate an inherited key.
  FOR EACH item IN source.items
    IF NO parent OF source DEFINES item.key
      out.items.append(item)
  RETURN out
```

**Invariants**

- Step 2's test is **key presence in a parent, not value equality**. A weapon's own line
  that *overrides* an inherited value is dropped along with one that merely repeats it,
  because the pull-down in step 1 already emitted the merged value for that key — but only
  if the rule asked for that key. A key the section overrides and the rule did not ask for
  is therefore **lost entirely**. That is the sharpest hazard in this file, and the reason
  the job description's key sets have to be written carefully rather than minimally.
- The parent list is copied onto the output section so the writer can restate the
  inheritance header. It is the only consumer of that field in the whole tool.

## `copy_params_ex` — the conversation

**Contract** — copies the wanted keys out of one parent section into the output, taking
each value from the *concrete* section's resolved view rather than from the parent, and
consulting the operator wherever the patch disagrees. Blocks on input. Records standing
answers so a key is asked about at most once per run.

```text
FUNCTION copy_params(out, parent, wanted_keys)
  FOR EACH item IN parent.items
    IF item.key NOT IN wanted_keys THEN CONTINUE

    # The value taken is the one the *concrete* section resolves to, which already
    # includes this parent's contribution and any nearer parent's override.
    chosen  <- value_of(previous, out.name, item.key)
    comment <- item.comment                 # the parent is where the documentation lives
    patched <- value_of(patch, parent.name, item.key)   # none if the patch is silent

    IF patched EXISTS AND patched != chosen
      IF item.key IN answered_yes  THEN chosen <- patched
      ELSE IF item.key IN answered_no THEN pass
      ELSE
        show(item.key, comment, chosen, patched)
        ANSWER <- read_one_key()            # this value / patch value / always either way
        IF ANSWER IS take_patch        THEN chosen <- patched
        IF ANSWER IS always_take_patch THEN answered_yes.add(item.key); chosen <- patched
        IF ANSWER IS always_keep       THEN answered_no.add(item.key)

    out.items.append(Item(item.key, chosen, comment))
```

**Notes**

- The patch is consulted under the **parent's** name while the value is read under the
  **concrete section's** name. That asymmetry is almost certainly unintended — a patch
  that overrides a value on the weapon itself rather than on its base section is never
  noticed — but it is what the tool does, and the shipped output was produced by it.
- The comment is taken from the parent because that is where a key is documented; a
  concrete section that repeats the key rarely repeats the explanation. The writer then
  emits each comment only once across the whole output file, so the documentation appears
  at its first use and not thirty times after.
- The operator's four answers are single keystrokes, deliberately: this is a conversation
  of a few hundred turns and anything longer than one key per turn is unusable. The
  *set* of answers is the decision; the input mechanism is not.

## `save_config_to_file` and `save_new_configs`

**Contract** — writes one output group to a file named for its rule, and then all groups in
turn. Each section is written as a header — name plus, when it has parents, the
inheritance list — followed by its items in column-aligned form, with a comment appended
the first time each distinct key appears in that file. Overwrites the destination. Fails
hard if the file cannot be opened.

```text
FUNCTION save_group(name, sections)
  seen_comments <- empty set<text>
  sink <- open_for_write(name)
  FOR EACH section IN sections
    header <- "[" + section.name + "]"
    IF section.parents NOT empty
      header <- header + ":" + join(section.parents, ", ")
    write_line(sink, header)
    FOR EACH item IN section.items
      line <- pad(item.key) + " = " + pad(item.value)
      IF item.comment EXISTS AND item.key NOT IN seen_comments
        line <- line + ";" + item.comment
        seen_comments.add(item.key)
      write_line(sink, line)
    write_blank_line(sink)
```

**Notes**

- This writer exists **instead of** the configuration object's own writer because that one
  does not emit the inheritance header — see [`xr_ini_ex.cpp`](xr_ini_ex.cpp.md). The
  duplication is not a decision, it is a gap; a rebuild should have one writer that can
  emit the header.
- Emitting each comment once per *file* rather than once per section is what makes the
  output readable: the same forty keys recur across thirty weapons, and repeating their
  documentation each time buries the numbers.
- The output file name is the rule's name verbatim, with no extension added and no
  directory: it is written relative to the working folder. Whether it ends in the
  configuration format's extension is the operator's business.
