# Shopping list format

* Proposal: [0016-shopping-list-format](0016-shopping-list-format.md)
* Authors: [Alexey Dubovskoy](https://github.com/dubadub)
* Status: **Awaiting review**

## Introduction

This proposal defines a plain-text format for shopping lists that can represent
recipe references with multipliers, free-hand ingredients with quantities, and
hierarchical nesting. The format is designed to be human-readable and editable
while supporting round-trip parsing and serialization.

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

Without a standard format, each application invents its own representation,
making it impossible to share shopping lists across tools or version-control them
alongside recipe files.

## Proposed solution

A line-oriented plain-text format where:

- **Recipe references** start with `./` followed by a path and an optional
  multiplier in braces: `./Breakfast/Easy Pancakes{2}`
- **Ingredients** are plain names with an optional quantity in braces:
  `olive oil{100%ml}`
- **Nesting** is expressed with 2-space indentation — children appear indented
  under their parent recipe
- **Comments** use `//` (line or inline)
- **Blank lines** are ignored

Example:

```
./Breakfast/Easy Pancakes{2}
  ./Sauces/Maple Syrup{1}
  butter{100%g}
  eggs{4}
free hand ingredient{4%l}
salt
// remember to check the pantry
```

This reads as: "Easy Pancakes scaled ×2 (which itself needs Maple Syrup ×1,
100 g butter, and 4 eggs), plus 4 l of a free-hand ingredient and some salt."

## Detailed design

### Grammar

A shopping list is a sequence of **lines**. Each line is one of:

| Line form | Meaning |
|---|---|
| `./path{multiplier}` | Recipe reference with multiplier |
| `./path` | Recipe reference without multiplier |
| `name{quantity%unit}` | Ingredient with quantity and unit |
| `name{quantity}` | Ingredient with quantity, no unit |
| `name` | Bare ingredient (no quantity) |
| `// ...` | Comment (full line) |
| *(empty)* | Ignored |

Inline comments are supported — everything after `//` on a line is discarded
before parsing.

More formally:

```
shopping_list  = { line }
line           = blank | comment | indented_item
blank          = WS* NL
comment        = WS* "//" any* NL
indented_item  = INDENT item NL
item           = recipe_ref | ingredient
recipe_ref     = "./" path [ "{" multiplier "}" ]
ingredient     = name [ "{" quantity "}" ]
path           = <any chars except '{' and '}'>
name           = <any non-empty chars except '{' and '}'>
multiplier     = NUMBER                     (* integer or decimal *)
quantity       = <any chars except '}'>      (* e.g. "4%l", "500%g", "2" *)
INDENT         = (" " " ")*                 (* 0, 2, 4, … spaces *)
```

### Indentation rules

- Each indentation level is exactly **2 spaces**.
- Only recipe references may have children.
- Children must be indented exactly one level deeper than their parent.
- An ingredient at the top level (indent 0) is a free-hand item not associated
  with any recipe.

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

Round-trip fidelity: `parse(serialize(list)) == list`.

## Effect on applications which use Cooklang

### CookCLI (terminal and web-server)

CookCLI should support reading and writing `.shopping` (or similar) files using
this format. A possible sub-command:

```
cook shopping-list create --recipe "Pancakes{2}" --recipe "Salad{1}" --add "milk{1%l}"
cook shopping-list show list.shopping
```

The web-server could expose an endpoint that returns a shopping list for selected
recipes.

### Mobile applications

Mobile apps should allow users to:

1. Select recipes and quantities, then generate a shopping list in this format
2. Display the hierarchical structure with recipe groupings
3. Add free-hand ingredients
4. Export / share the list as plain text
5. Check off items while shopping

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
