# Shopping list format

* Proposal: [0016-shopping-list-format](0016-shopping-list-format.md)
* Authors: [Alexey Dubovskoy](https://github.com/dubadub)
* Status: **Awaiting review**

## Introduction

This proposal defines a plain-text format for shopping lists that can represent
recipe references with multipliers, free-hand ingredients with quantities, and
hierarchical nesting. It also defines a companion check file for tracking which
ingredients have been acquired. The format is designed to be human-readable and
editable while supporting round-trip parsing and serialization.

## Motivation

Cooklang recipes define ingredients inline, but there is currently no standard
format for representing a composed shopping list — the artifact that results from
selecting one or more recipes and combining their ingredients for a shopping
trip.

Users need a way to:

1. Compose a shopping list from multiple recipes, each potentially scaled to a
   different number of servings
2. Add free-hand ingredients that aren't part of any recipe (e.g. pantry
   restocking)
3. Organize items hierarchically so it's clear which ingredients belong to which
   recipe
4. Edit the list by hand in any text editor before heading to the store
5. Check off items while shopping, with support for undo

Without a standard format, each application invents its own representation,
making it impossible to share shopping lists across tools or version-control them
alongside recipe files.

## Proposed solution

A line-oriented plain-text format consisting of two hidden files per directory:

- **`.shopping-list`** — the shopping list definition
- **`.shopping-checked`** — an append-only log of checked/unchecked ingredients

The shopping list definition uses:

- **Recipe references** starting with `./` followed by a path and an optional
  multiplier in braces: `./Breakfast/Easy Pancakes{2}`
- **Ingredients** as plain names with an optional quantity in braces:
  `olive oil{100%ml}`
- **Nesting** expressed with 2-space indentation — children appear indented
  under their parent recipe
- **Comments** using `--` (line or inline) and `[- -]` (block), consistent with
  Cooklang recipe syntax
- **Blank lines** are ignored

Example `.shopping-list`:

```
./Plans/3 Day Plan I
  ./Breakfast/Mexican Style Burrito{2}
    ./Components/Guacamole{2}
    ./Components/Beans{2}
  ./Salads/Boring{2}
  ./Slowcooker/Slow-cooker beef stew{0.5}
olive oil{4%l}
salt
-- remember to check the pantry
```

This reads as: "3 Day Plan I containing Mexican Style Burrito ×2 (which itself
needs Guacamole ×2 and Beans ×2), Boring Salad ×2, and half a batch of beef
stew — plus 4 l of olive oil and salt as free-hand items."

Nested items are always recipe references. Plain ingredients only appear at the
top level (indent 0) as free-hand items not associated with any recipe.

Example `.shopping-checked`:

```
+ salt
+ avocados
+ olive oil
- avocados
+ avocados
```

Final state: salt (checked), avocados (checked), olive oil (checked).

## Detailed design

### Shopping list grammar

A shopping list (`.shopping-list`) is a sequence of **lines**. Each line is one
of:

| Line form | Meaning |
|---|---|
| `./path{multiplier}` | Recipe reference with multiplier (any indent level) |
| `./path` | Recipe reference without multiplier (any indent level) |
| `name{quantity%unit}` | Ingredient with quantity and unit (top-level only) |
| `name{quantity}` | Ingredient with quantity, no unit (top-level only) |
| `name` | Bare ingredient (no quantity, top-level only) |
| `-- ...` | Line comment |
| `[- ... -]` | Block comment |
| *(empty)* | Ignored |

Inline comments are supported — everything after `--` on a line is discarded
before parsing. Block comments `[- ... -]` may span multiple lines.

Only recipe references may be nested (indented). Plain ingredients are always
top-level (indent 0) free-hand items.

More formally:

```
shopping_list  = { line }
line           = blank | comment | top_level_item | indented_ref
blank          = WS* NL
comment        = WS* "--" any* NL
top_level_item = (recipe_ref | ingredient) NL
indented_ref   = INDENT recipe_ref NL
recipe_ref     = "./" path [ "{" multiplier "}" ]
ingredient     = name [ "{" quantity "}" ]
path           = <any chars except '{' and '}'>
name           = <any non-empty chars except '{', '}', and not starting with "./">
multiplier     = NUMBER                     (* integer or decimal *)
quantity       = <any chars except '}'>      (* e.g. "4%l", "500%g", "2" *)
INDENT         = (" " " ")+                 (* 2, 4, 6, … spaces *)
```

### Check file grammar

A check file (`.shopping-checked`) is a sequence of **lines**:

| Line form | Meaning |
|---|---|
| `+ name` | Ingredient checked (acquired) |
| `- name` | Ingredient unchecked |
| *(empty)* | Ignored |

`name` is the ingredient name only — no quantities, no braces.

The last entry for a given ingredient name determines its current state. If the
last entry is `+`, the ingredient is checked; if `-`, it is unchecked.

More formally:

```
check_file     = { check_line }
check_line     = blank | check_entry
blank          = WS* NL
check_entry    = ("+" | "-") " " name NL
name           = <any non-empty chars>
```

### Indentation rules

- Each indentation level is exactly **2 spaces**.
- Only recipe references may be indented (nested).
- Children must be indented exactly one level deeper than their parent.
- Multiple levels of nesting are supported — a menu can contain recipes, which
  can contain sub-recipes (e.g. a meal plan → a burrito recipe → guacamole).
- Plain ingredients only appear at the top level (indent 0) as free-hand items
  not associated with any recipe.

### Check semantics

The `.shopping-checked` file is an append-only log. Applications append `+ name`
when the user checks an ingredient and `- name` when the user unchecks it. The
file is never edited in-place except during compaction.

An ingredient's checked state is determined by its **last entry** in the log. If
an ingredient has no entry, it is unchecked.

### Compaction

Compaction rebuilds the `.shopping-checked` file to its minimal form. A
conforming compactor must:

1. **Replay** the log top-to-bottom, determining the final state for each
   ingredient (last entry wins)
2. **Reconcile** against the current `.shopping-list` — drop any entries for
   ingredients that no longer appear in the shopping list
3. **Rewrite** the file with one `+ name` line per checked ingredient
   (unchecked ingredients are simply omitted)

After compaction the file contains only `+` entries. Compaction is
**idempotent**: compacting an already-compacted file produces the same file.

The spec defines the semantics; when to compact is left to applications.

### File naming and visibility

Both files live in the same directory. The full filenames are `.shopping-list`
and `.shopping-checked` — there is exactly one shopping list per directory.

On Unix-like systems these files are hidden by convention (dot-prefix). On
Windows, applications should set the hidden file attribute to keep them out of
normal directory listings.

### Multiplier

The multiplier is a positive number (integer or decimal) enclosed in braces
immediately after the recipe path. It represents how many times the recipe should
be scaled. Fractional multipliers like `{0.5}` are supported.

### Quantity format

Ingredient quantities reuse the Cooklang `%` separator between the numeric value
and the unit: `{500%g}`, `{4%l}`, `{2}`. The quantity string is preserved
as-is — parsing the value/unit split is left to applications.

### Serialization

A conforming serializer must:

1. Write recipe references as `./path{multiplier}` (omit braces when there is no
   multiplier; write integer multipliers without a decimal point)
2. Write ingredients as `name{quantity}` (omit braces when there is no quantity)
3. Indent children with 2 spaces per nesting level
4. Terminate every line with a newline
5. Write the shopping list to `.shopping-list`
6. Write check entries to `.shopping-checked` using `+ name` / `- name` syntax

Round-trip fidelity: `parse(serialize(list)) == list`.

## Effect on applications which use Cooklang

### CookCLI (terminal and web-server)

CookCLI should support reading and writing `.shopping-list` and
`.shopping-checked` files. Possible sub-commands:

```
cook shopping-list create --recipe "Pancakes{2}" --recipe "Salad{1}" --add "milk{1%l}"
cook shopping-list show
cook shopping-list check salt
cook shopping-list uncheck salt
cook shopping-list compact
```

The web-server could expose an endpoint that returns a shopping list for selected
recipes, along with checked state.

### Mobile applications

Mobile apps should allow users to:

1. Select recipes and quantities, then generate a `.shopping-list` file
2. Display the hierarchical structure with recipe groupings
3. Add free-hand ingredients
4. Export / share the list as plain text
5. Check off items while shopping (appending to `.shopping-checked`)
6. Display checked state by replaying the check log
7. Compact the check file periodically or on user action

## Edge cases

### Case-insensitive matching

Ingredient names in the check file are matched against the shopping list
**case-insensitively**. `+ Salt`, `+ salt`, and `+ SALT` all refer to the same
ingredient.

### Global check semantics

A check entry applies to the ingredient **everywhere** it appears in the
shopping list, regardless of which recipe it belongs to. If `butter` appears
under both `./Pancakes` and `./Cookies`, then `+ butter` checks it off as a
single item to buy.

### Whitespace normalization

Leading and trailing whitespace is trimmed from ingredient names in both files
before matching. The check entry prefix is always exactly `+` or `-` followed by
a single space.

### Line endings

Parsers must accept both `\n` (Unix) and `\r\n` (Windows) as line terminators.
Serializers should write the platform default.

### Missing files

- `.shopping-list` without `.shopping-checked` — everything is unchecked.
- `.shopping-checked` without `.shopping-list` — the check file has no effect.
  Compaction would produce an empty file.
- An empty `.shopping-checked` file is equivalent to a missing one.

The `.shopping-list` is always the source of truth.

## Alternatives considered

1. **JSON or YAML format** — more structured but harder to read and edit by hand
   in a plain-text editor. The line-oriented format is more consistent with
   Cooklang's philosophy of human-readable plain text.

2. **Flat ingredient list with no recipe grouping** — simpler but loses the
   relationship between recipes and their ingredients, making it harder to adjust
   quantities or remove a recipe from the list.

3. **Reusing Cooklang recipe syntax** — recipe files describe cooking steps, not
   shopping items. Overloading the recipe format would conflate two different
   concerns.

## Acknowledgments

Thanks to the Cooklang community for feedback on recipe composition workflows.
