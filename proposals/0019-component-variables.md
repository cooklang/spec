# Named component variables

* Proposal: [0019-component-variables](0019-component-variables.md)
* Authors: [Luka Hummel](https://github.com/LukaHummel)
* Review Manager: TBD
* Status: **Awaiting review**

## Introduction

This proposal introduces recipe-scoped, named component variables. Ingredient
variables let an author provide an ordered list of alternative ingredients,
each with its own total amount and preparation, and use one stable name in the
recipe steps. Inline amounts can allocate that total between steps, allowing
language servers and linters to warn when the allocations do not add up.
Cookware variables let an author define indexed physical items, so a
recipe can distinguish two otherwise identical pots without calling them
"Pot 1" and "Pot 2" in its prose. Applications can resolve ingredient choices,
preserve cookware identity while cooking, and aggregate identical cookware in
the equipment list.

Related discussion threads:

* [Syntax for suggesting alternative ingredients](https://github.com/cooklang/spec/discussions/68)
* [Recipe variants](https://github.com/cooklang/spec/discussions/75)

## Motivation

Cooklang can currently express only the ingredient selected by the recipe
author:

```cooklang
Dissolve @fresh yeast{10%g} in water.
```

The author can describe a substitute in prose or a comment:

```cooklang
Dissolve @fresh yeast{10%g} in water. -- Or use 5 g dry yeast.
```

This preserves the recipe's readability, but the substitute is not
machine-readable. An application cannot reliably select dry yeast, replace the
name and amount in the step, or use the selected option in a shopping list.
Writing alternatives next to each other also becomes repetitive when an
ingredient is mentioned in more than one step.

Recipes may also divide one ingredient between steps, but Cooklang has no
declaration for the expected total. Tooling cannot distinguish a deliberate
partial use from an amount accidentally omitted elsewhere, nor warn that the
inline uses exceed the amount intended for the whole recipe.

Cookware has a related identity problem. A recipe which uses two pots can write
`#pot{2}`, but that does not identify which pot a later step refers to. Authors
can write "Pot 1" and "Pot 2", but those labels become part of the cookware
names and prevent applications from presenting the requirement as `2× pot` or
`pot (2)`. The labels also carry no structured identity that a cooking view can
consistently render with colors or badges.

Both problems need a stable recipe-scoped name. For ingredients, the name
identifies a choice whose selected value is used throughout the recipe. For
cookware, the name and an index identify a physical item whose repeated uses do
not increase the amount of cookware required.

This proposal intentionally addresses component choices and identity, not
whole-recipe variants. A variant system may also replace steps, sections, and
multiple ingredients together; that broader problem remains separate.

## Proposed solution

Add a canonical `variables` key to YAML front matter. Each entry is typed as an
`ingredient` or `cookware` variable.

Ingredient variables contain an ordered `options` list. Each option declares
the ingredient that can fill the variable and may declare its own total amount,
preparation, and explanatory note. The first option is the default.

Cookware variables contain an ordered `items` list. Each position identifies
one physical item, even when multiple positions have the same cookware name.

```cooklang
---
variables:
  yeast:
    type: ingredient
    options:
      - name: fresh yeast
        amount: 10%g
        note: preferred
      - name: dry yeast
        amount: 5%g
        note: if fresh yeast is unavailable

  pots:
    type: cookware
    items:
      - pot
      - pot
---

Dissolve @yeast{4%g} in #pots[0].

Add the remaining @yeast{6%g} to #pots[1], then clean #pots[0].
```

`@yeast` uses the selected ingredient option everywhere it appears. Without an
explicit client choice, it resolves to `10 g fresh yeast`. Selecting option `1`
resolves it to `5 g dry yeast` instead. Each option's declared `amount` is its
total amount for the whole recipe. The two inline amounts allocate 40% and 60%
of that total, so selecting dry yeast renders them as `2 g` and `3 g`.

`#pots[0]` and `#pots[1]` are distinct physical pots. Referring to `#pots[0]`
again does not require a third pot. An application can therefore preserve both
identities in the recipe steps while displaying the equipment requirement as
`2× pot`.

## Detailed design

The keywords **MUST**, **MUST NOT**, **SHOULD**, and **SHOULD NOT** in this
proposal describe requirements for interoperable implementations.

### Variable declarations

The `variables` value MUST be a YAML mapping. Each mapping key is a variable
identifier and each value is a variable declaration.

A variable identifier MUST match:

```text
[A-Za-z_][A-Za-z0-9_]*
```

Identifiers are case-sensitive. A variable identifier is reserved across all
component types in its recipe. Two variables therefore cannot use the same
identifier, even if their types would be different.

Declarations may appear in any order. Their order has no semantic meaning.

#### Ingredient variables

An ingredient variable has this shape:

```yaml
variable_name:
  type: ingredient
  options:
    - name: first ingredient
      amount: 100%g
      preparation: finely chopped
      note: optional explanation
    - name: second ingredient
      amount: 80%g
```

`options` MUST be an ordered sequence containing at least two entries. Each
entry MUST be a mapping with a non-empty string `name` and MAY contain:

* `amount`: the option's total amount for the whole recipe, using the existing
  Cooklang amount syntax without the surrounding braces.
* `preparation`: a non-empty string equivalent to Cooklang's shorthand
  preparation text, without the surrounding parentheses.
* `note`: a non-empty, human-readable explanation of when or why the option may
  be chosen.

The `name` MUST NOT include `@` or Cooklang component delimiters. `note` is
display information only. It MUST NOT activate recipe variants or conditionally
include steps.

Option indexes are zero-based. Option `0` is always the default; there is no
separate `default` field. Reordering options therefore changes both their
indexes and the default choice.

Every option resolves to exactly one ingredient. An option cannot contain a
list or group of ingredients.

An ingredient variable can be referenced with inline amounts only when every
option declares a single, positive numeric `amount`. The option amounts may use
different quantities or units because inline allocations are transferred
proportionally, as described below.

#### Cookware variables

A cookware variable has this shape:

```yaml
variable_name:
  type: cookware
  items:
    - pot
    - pot
```

`items` MUST be an ordered sequence containing at least one non-empty string.
Each string is a cookware name and MUST NOT include `#` or Cooklang component
delimiters. Duplicate names are valid and represent separate physical items.

Item indexes are zero-based. The tuple `(variable identifier, item index)` is
the stable identity of an item within a recipe.

### References in recipe text

Ingredient variables deliberately use the existing readable form of a
one-word ingredient:

```cooklang
Add @yeast to the bowl.
```

After parsing front matter, an ingredient whose complete name matches a
declared variable identifier is a variable reference. It MUST resolve to the
declared variable and MUST have type `ingredient`.

Cookware items use an index after the declared variable name:

```cooklang
Heat #pots[0] while #pots[1] cools.
```

The grammar can be described as:

```ebnf
variable identifier          = ( letter | "_" ), { letter | digit | "_" } ;
variable index               = digit, { digit } ;
ingredient variable reference = "@", variable identifier ;
cookware variable reference   = "#", variable identifier,
                                "[", variable index, "]" ;
```

In this grammar, `letter` is one of `A-Z` or `a-z`, and `digit` is one of
`0-9`.

An ingredient variable reference MAY contain an inline amount. It allocates a
portion of the declared total rather than replacing or adding to that total:

```cooklang
Bloom @yeast{4%g} in water.

Add the remaining @yeast{6%g} to the dough.
```

An ingredient variable reference MUST NOT have an empty amount block,
preparation, or another component suffix. Cookware variable references MUST
NOT have any component suffix. These examples are invalid:

```cooklang
Add @yeast{}.
Add @yeast(room temperature).
Heat #pots[0]{2}.
```

A reference to a declared name using the wrong component marker is an error.
For example, `#yeast[0]` is invalid when `yeast` is an ingredient variable, and
`@pots` is invalid when `pots` is a cookware variable. A cookware variable used
without an index, such as `#pots`, is also invalid.

If an ingredient name is not declared in `variables`, it remains an ordinary
ingredient. Similarly, text which resembles an indexed cookware reference but
does not name a declared cookware variable retains the behavior it would have
without this proposal. Recipes without `variables` are unaffected.

### Parsing and validation

Implementations MUST interpret a recipe in two stages:

1. Parse the YAML front matter and validate all variable declarations.
2. Parse recipe components and resolve names against the validated variable
   map.

An implementation MUST report an error for:

* An invalid variable identifier.
* An unknown variable `type`.
* A missing or empty `options` or `items` sequence.
* An ingredient variable with fewer than two options.
* A malformed option, cookware name, or Cooklang amount.
* An ingredient variable reference with an empty amount block, preparation, or
  another unsupported suffix.
* A cookware variable reference with any component suffix.
* An inline allocation which is not a single, positive numeric amount.
* An inline allocation when any option lacks a single, positive numeric total.
* An inline allocation whose unit does not exactly match the default option's
  unit, including one amount being unitless while the other has a unit.
* An inline allocation or alternative total whose fixed-versus-scalable marker
  differs from the default total.
* A declared variable used with the wrong component marker.
* A cookware variable used without an index.
* A cookware index outside the declared `items` sequence.
* A client ingredient selection outside the declared `options` sequence.

An implementation SHOULD warn when an ingredient or cookware variable is never
referenced. It SHOULD also warn for each cookware item which is declared but
never referenced. These warnings do not make the recipe invalid.

### Inline amount allocation and total validation

An ingredient option's `amount` is authoritative: it is the total amount of
that option required by the whole recipe. It is not an amount per reference.
The resolved ingredient and shopping lists MUST use this declared total even
when inline allocations do not add up.

An ingredient variable reference without an amount is a mention of the logical
ingredient and does not allocate any of its total. A reference with an amount
is an allocation. Every occurrence with an inline amount participates in total
validation.

Inline amounts are authored relative to option `0`. Each inline allocation MUST
use the same unit string as the default total, or both values MUST be unitless.
Requiring one unit avoids making this feature depend on a unit-conversion table
which is not part of this proposal. Let:

* `T0` be option `0`'s declared total.
* `Ai` be one inline allocation.
* `Si = Ai / T0` be that allocation's share of the total.

When option `n` is selected, the amount rendered for that occurrence is
`Si * Tn`, expressed using option `n`'s declared unit. The selected option's
unit does not need to be compatible with option `0` because only the
dimensionless share is transferred.

For example:

```cooklang
---
variables:
  yeast:
    type: ingredient
    options:
      - name: fresh yeast
        amount: 10%g
      - name: dry yeast
        amount: 5%g
---

Bloom @yeast{4%g} in water.

Add @yeast{6%g} to the dough.
```

With the default option, the occurrences render as `4 g` and `6 g`. With dry
yeast selected, their shares remain 40% and 60%, so they render as `2 g` and
`3 g`. The ingredient and shopping lists show the selected total: either `10 g
fresh yeast` or `5 g dry yeast`.

Language servers and linters SHOULD sum all inline allocations
for each ingredient variable. They SHOULD warn when the sum differs from `T0`
and include the declared total, allocated total, and difference in the
diagnostic. For example, allocations of `4 g` and `5 g` against a declared
`10 g` total should report that `1 g` remains unallocated. Allocations above the
total should report the excess.

Implementations SHOULD use exact decimal or rational arithmetic where
possible, and MUST NOT emit a mismatch warning solely because of binary
floating-point or display-rounding error.

The mismatch is a warning, not a parse error. This permits an author to edit an
incomplete recipe while retaining an authoritative total. If a variable has no
inline allocations, implementations MUST NOT emit an add-up warning; its
unquantified references are mentions and its declaration alone defines the
total.

### Ingredient selection and resolution

Ingredient selection is client state, not recipe metadata. A client supplies a
mapping from ingredient variable identifiers to zero-based option indexes. For
example:

```json
{
  "yeast": 1
}
```

If the mapping omits an ingredient variable, the client MUST select option `0`.
If it provides an invalid index, the client MUST report an error and MUST NOT
silently use the default.

All references to an ingredient variable identify one recipe-wide logical
ingredient. Repeating an unquantified reference can mention the selected
ingredient in multiple steps, but it does not add or multiply the option's
total:

```cooklang
Bloom @yeast in the water.

Make sure @yeast has started to foam before continuing.
```

The selected option contributes once to the recipe's ingredient list and once
to shopping-list calculation. Quantified references divide that total between
occurrences; they never increase the authoritative total.

The existing serving-scaling rules apply independently to every option total
and every inline allocation. Scaling both by the same factor preserves each
allocation share. All option totals and inline allocations in one variable
MUST therefore use the same fixed-versus-scalable behavior. Only the selected
option is included in resolved ingredient or shopping-list output.

An option may omit `amount`, matching an ordinary ingredient without a
quantity:

```cooklang
---
variables:
  garnish:
    type: ingredient
    options:
      - name: parsley
      - name: cilantro
---

Finish with @garnish to taste.
```

### Cookware identity and aggregation

Only cookware items referenced in recipe text contribute to the required
cookware list. Repeating the same `(variable identifier, item index)` does not
increase the requirement:

```cooklang
Heat the sauce in #pots[0].

Return the sauce to #pots[0].
```

This requires one pot. By contrast:

```cooklang
Heat the sauce in #pots[0].

Boil the pasta in #pots[1].
```

This requires two pots. Applications MAY aggregate identical cookware names as
`2× pot`, `pot (2)`, or another locale-appropriate representation. They MUST
retain the variable identifier and index on step references so the two items
can still be distinguished.

Logical identities remain distinct across variables. If two referenced items
from different variables are both named `pot`, both contribute to the required
count. Authors who intend to reuse the same physical pot must reference the
same variable and index.

### Normalized data model

Parser APIs and serialized output SHOULD retain declarations and symbolic
references rather than discarding unselected options during parsing. The exact
host-language representation can vary, but it should expose the equivalent of:

```text
VariableDefinition =
  IngredientVariable {
    type: "ingredient",
    options: IngredientOption[],
    default: 0
  }
  | CookwareVariable {
    type: "cookware",
    items: string[]
  }

IngredientVariableRef {
  type: "ingredient",
  variable: string,
  amount?: Amount
}

CookwareVariableRef {
  type: "cookware",
  variable: string,
  index: integer
}
```

`IngredientOption` contains `name` and the parsed values of any total `amount`,
`preparation`, and `note`. `default: 0` is derived from ordering and need not be
stored in the source metadata. An `IngredientVariableRef` retains its parsed
inline allocation, when present. Resolvers may derive and cache its
dimensionless share, but serialized parser output should preserve the authored
amount.

A resolver consumes this selection-neutral representation plus a choice map
and produces the effective step components, ingredient list, shopping list,
and cookware summary.

### Backward compatibility

Recipes without a valid `variables` declaration keep their current semantics.
The front matter remains valid YAML, so applications which preserve unknown
metadata can retain the declarations even before they implement this feature.

Bare ingredient references have an intentional compatibility limitation. A
client which does not implement variables will treat `@yeast` as a literal
ingredient named `yeast`; it cannot recover `fresh yeast`, `dry yeast`, or
their amounts. An older client may likewise parse `#pots[0]` as `#pots`
followed by plain text. Authors should therefore expect recipes using this
feature to require a supporting client for correct ingredient, cookware, and
shopping-list output.

## Effect on applications which use Cooklang

Applications must preserve the distinction between parsing a recipe and
resolving a user's ingredient choices. They should not permanently rewrite the
recipe when a user changes a choice.

Ingredient lists should show only the selected option in their normal resolved
view. Applications may additionally expose all alternatives, including each
option's amount, preparation, and note. Changing a selection must update every
reference, its proportionally resolved inline amount, the ingredient summary,
and shopping-list output consistently. The summary uses the selected option's
declared total rather than calculating a replacement total from the inline
allocations.

Cooking views should preserve cookware identity across steps. Colors, badges,
or generated labels may be used to distinguish items, but color must not be the
only identity cue in an accessible interface.

### Language servers and linters

Language servers and linters should validate inline allocation totals while an
author edits a recipe. For each variable with at least one quantified
reference, they should:

1. Verify that allocations use option `0`'s unit.
2. Sum every quantified occurrence.
3. Compare the sum with option `0`'s declared total.
4. Report a warning on the variable declaration and relevant references when
   the sum is below or above the total.

Diagnostics should state the declared total, allocated total, and remaining or
excess amount. Tooling should update or remove the warning as references are
edited. It should not emit a total warning for a variable used only through
unquantified mentions.

### CookCLI (terminal and web-server)

CookCLI's JSON output should include the complete variable declarations and
the variable identifier or `(variable, index)` on symbolic step items. An
ingredient reference should retain its authored inline allocation when one is
present.

Human-readable output should resolve missing choices to option `0`. Existing
commands do not require a new sub-command. Commands which render a recipe or
generate a shopping list may accept a repeatable option such as:

```console
cook recipe --choice yeast=1 recipe.cook
```

The exact placement of the option can follow CookCLI's existing command-line
conventions, but its value represents the choice map defined above. Invalid
variable names or indexes must produce a clear error rather than silently
falling back to the default.

The web server should expose the same declarations and choice-map semantics in
its API and provide ingredient selectors in rendered recipes. Step rendering
should show the allocation scaled to the selected option, while ingredient and
shopping summaries continue to show the selected declared total.

### Mobile applications

Mobile applications should add a selector for each referenced ingredient
variable. A selector should display the name, amount, preparation, and note for
each option. Selecting an option should immediately update:

* Every occurrence in the cooking steps.
* The ingredient summary.
* Any shopping list derived from the recipe.

Quantified occurrences should show their proportionally resolved amounts. For
example, changing from `10 g fresh yeast` to `5 g dry yeast` changes a `4 g`
allocation to `2 g`, while the ingredient summary changes from `10 g` to `5 g`.

Cooking steps should use the stable cookware identity to distinguish items
such as `pots[0]` and `pots[1]`. The equipment summary may aggregate those
items as two pots while step views use colors, badges, or generated accessible
labels to show which pot is meant.

## Alternatives considered

### Explicit variable sigil

References could use a dedicated sigil, for example `@$yeast` and
`#$pots[0]`. This would be unambiguous without reading front matter and would
make missing declarations easier to diagnose. Bare names were chosen because
they keep the recipe text closer to natural Cooklang prose and avoid adding a
second marker to every reference. The tradeoff is weaker behavior in clients
which do not implement variables.

### Inline alternatives

Discussion #68 proposes pipe-separated alternatives such as:

```cooklang
Add @fresh yeast{10%g}|@dry yeast{5%g}.
```

This is concise when alternatives occur once. It requires the alternatives to
appear together in the prose, duplicates them when the logical ingredient is
mentioned later, and does not provide stable indexed cookware identity.

### Labeled or grouped alternatives

Alternatives can be associated with a shared inline label. This gives authors
more control over natural sentence flow and can support grouped replacements,
but it distributes the definition across the recipe. A front-matter variable
provides one authoritative list of choices and amounts which clients can
validate before resolving any step.

### Separate metadata namespaces

The proposal could add `ingredient_variables` and `cookware_variables` keys.
One typed `variables` map was chosen because it keeps all recipe-scoped names
in one namespace and can be extended with other component types in a future
proposal.

### Top-level ingredient and cookware maps

Using top-level `ingredients` and `cookware` metadata would be shorter, but
those broad names are likely to collide with existing application-specific
metadata. Nesting definitions under `variables` makes the new semantics
explicit.

### Explicit option identifiers and defaults

Each option could declare a stable identifier and one option could be marked as
the default. That survives reordering but adds authoring overhead. Ordered
zero-based indexes were chosen because YAML already preserves sequence order,
the first alternative naturally represents the author's preference, and the
model matches cookware indexing.

### Multi-ingredient replacements

Some substitutions replace one ingredient with several, such as salted butter
with butter and salt. This proposal restricts each option to one ingredient.
Bundles require additional rules for grouped quantities, repeated references,
shopping-list aggregation, and prose rendering, so they should be considered
separately.

### Literal amounts for every alternative

An inline amount could be applied literally regardless of the selected option.
That makes a recipe with `10 g fresh yeast` and `5 g dry yeast` inconsistent:
inline allocations which add up for one option cannot also add up for the
other. Authors could instead repeat an amount for every option at every
reference, but that would be verbose and would duplicate the front-matter
choice structure throughout the steps. Proportional allocations let authors
write and validate the default amounts once while preserving the declared total
of every alternative.

### Whole-recipe variants

A recipe variant may replace steps, sections, ingredients, and cookware as a
coherent version such as "vegan" or "low effort". Named component variables do
not conditionally include prose or coordinate choices across variables. This
keeps simple substitutions independent from the broader variants work
discussed in discussion #75.

### Structured front matter precedent

The proposed
[Baker's Percentages and Named Ingredient Bases](https://github.com/spiicy-sauce/cooklang-spec/blob/51220fc26f26cb871d7193dce5eb505520a49bd1/proposals/0018-bakers-percentages.md)
feature similarly declares structured data in front matter and gives existing
component syntax additional semantic meaning. Named component variables use
the same general separation between declaration, parsing, and resolution, but
solve component selection and physical identity rather than relative amounts.

## Normative scenarios

Implementations should cover at least these scenarios:

1. With no choice map, a variable whose option `0` is `10 g fresh yeast`
   contributes that total once to ingredient and shopping lists.
2. Inline allocations of `4 g` and `6 g` render unchanged for the default
   option and produce no total warning.
3. With `{ "yeast": 1 }` selecting `5 g dry yeast`, the same occurrences
   render as `2 g` and `3 g`, while the ingredient list shows the `5 g` total.
4. Inline allocations of `4 g` and `5 g` against a `10 g` default total produce
   a warning that `1 g` remains unallocated; allocations above `10 g` produce
   an excess warning.
5. Mentioning unquantified `@yeast` in multiple steps contributes its declared
   total only once and does not participate in allocation validation.
6. A variable with no quantified references produces no add-up warning.
7. An inline allocation is rejected when it is not a single positive amount,
   when an option lacks a single positive total, when its unit differs from the
   default total's unit, or when scaling markers differ.
8. Scaling a recipe scales every option total and inline allocation by the same
   factor, preserving their shares.
9. Referencing `#pots[0]` repeatedly requires one pot.
10. Referencing `#pots[0]` and `#pots[1]` requires two distinct pots which may
    be aggregated in the equipment summary.
11. A declared but unreferenced cookware item does not contribute to the summary
    and produces a warning.
12. An ingredient variable whose options omit `amount` resolves like an
    ordinary unquantified ingredient when all its references are also
    unquantified.
13. Wrong component types, missing cookware indexes, indexes outside the
    declared sequences, invalid client selections, and unsupported suffixes
    produce errors.
14. A recipe without `variables` parses and resolves exactly as it did before
    this proposal.

## Acknowledgments

Thanks to the participants in
[discussion #68](https://github.com/cooklang/spec/discussions/68) for exploring
inline, labeled, optional, and grouped ingredient alternatives, and to the
participants in
[discussion #75](https://github.com/cooklang/spec/discussions/75) for clarifying
the boundary between substitutions and full recipe variants. Thanks also to
[Teddy Cleveland](https://github.com/spiicy-sauce) for the structured
front-matter approach in the Baker's Percentages proposal.
