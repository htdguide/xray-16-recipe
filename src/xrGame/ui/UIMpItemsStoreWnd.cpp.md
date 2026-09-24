# src/xrGame/ui/UIMpItemsStoreWnd.cpp

> The store's category tree: its *shape* comes from the layout document and its *contents* from
> the configuration section for one team, which is why the two are loaded separately.

**Needs** — [`UIMpItemsStoreWnd.h`](UIMpItemsStoreWnd.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`UITabButtonMP.h`](UITabButtonMP.h.md) · [`Restrictions.h`](Restrictions.h.md)
**Used by** — [`UIMpItemsStoreWnd.h`](UIMpItemsStoreWnd.h.md)
**Tier floor** — T3.

## Purpose

The buy screen browses a tree of categories — *weapons* → *rifles* → the rifles themselves —
and this file is that tree. The interesting decision is the **split of authorship**: the tree's
shape and the button that selects each branch are authored in the *layout document*, while
which items live at a leaf comes from a *configuration section* named after the team. One
shape, two fillings, and adding a weapon to a team is a configuration edit alone.

## State

```text
RECORD StoreNode
  name           : text                # the category's identifier, also the button's id
  button_element : text                # which layout element supplies this node's button
  parent         : optional<StoreNode>
  children       : list<StoreNode>
  items_in_group : list<text>          # item sections; non-empty only at a leaf
  button         : optional<TabButton> # built for every node below the root that names one

RECORD StoreHierarchy
  root          : StoreNode
  current_level : StoreNode            # the browsing cursor
  team_idx      : int                  # which team's store this is
```

**Invariants**

- **A node either has children or has items, never both.** The item loader only descends into a
  node with children and only reads items at a node without them; the search asserts the same.
  This is what makes the tree a clean two-phase structure — categories above, wares at the leaf.
- The root has no button. Buttons exist from depth one downward, and the depth-one buttons
  become the screen's top-level tab strip while deeper ones are laid out by the layout document
  as the browse level changes.
- A node's *name* is what the cursor moves by and what a button carries as its identity, so
  names must be unique among siblings. Nothing enforces it; a duplicate makes the first match
  win.
- Every item named at a leaf must also belong to a *restriction group* — the table that caps
  how many of a kind may be carried. A leaf item with no group is a data error the loader
  catches.

## `Init` and `LoadLevel`

**Contract** — read a nested `level` element tree from a subtree of the layout document, one
node per element, recursively. Each node takes its name and its button element from attributes;
a node below the root that names a button element builds that button from the *document root*,
tags it with the node's name, and orients it horizontally or vertically according to that
element's own attribute. The cursor starts at the root.

```text
FUNCTION load_level(doc, index, node, depth)
  node.name           <- attribute "name"   of level[index]
  node.button_element <- attribute "btn_ref" of level[index]
  IF depth > 0 AND node.button_element is non-empty THEN
    node.button <- tab_button built from the DOCUMENT ROOT at node.button_element
    node.button.id <- node.name
    node.button.horizontal <- that element's "horz_al" attribute is 1
  FOR i IN 0..count of nested "level" elements under level[index]
    child <- new node with parent = node
    node.children.append(child)
    load_level(doc, i, child, depth + 1)
```

**Notes** — the button is built against the document's *root*, not against the current subtree,
while the tree's shape is read relative to the subtree. So the hierarchy declares the tree and
*refers by name* to button definitions that live elsewhere in the same document. That
indirection is what lets two branches share one button appearance, and it is the reason the
reader has to save and restore its position twice per node.

## `InitItemsInGroup`

**Contract** — walk the tree; at every leaf, read the configuration key named after that leaf
from the given section and split its value into item sections. At the root, additionally record
the team index from that section. Each item is checked to belong to a restriction group.

**Notes** — the leaf's *name* is a configuration key. So the layout document's category names
and the configuration section's key names are one shared vocabulary, and renaming a category in
the layout silently empties it. That coupling is the price of the authorship split, and a
rebuild should keep it explicit rather than trying to hide it.

The guard at the root checks that the team section does **not** carry a team-name line, while
its failure message says the opposite. One of the two is wrong; the effect in a shipping build
is nothing, since the check is compiled out.

## `FindItem`

**Contract** — find the leaf that sells a given item section, searching the whole tree from the
root or from a given node. Returns the leaf, not the item.

**Notes** — the recursion returns the *child it descended into* rather than the leaf the match
was found at, so a match two levels down reports the intermediate node. Every caller only tests
the result for existence, so the difference has never mattered; a rebuild should return the
leaf.

## The cursor

**Contract** — `Reset` returns to the root. `MoveDown` descends into the named child and fails
loudly if there is none. `MoveUp` ascends, reporting false at the root. `CurrentIsRoot` is what
the screen uses to decide whether its *back* button is live and whether to show the tab strip
rather than a category.

## The node queries

**Contract** — `HasItem` and `GetItemIdx` search a leaf's item list by section, the latter
returning the position or a not-found marker. That position is what the buy screen turns into a
keyboard accelerator, so the order of items within a leaf is player-visible.
