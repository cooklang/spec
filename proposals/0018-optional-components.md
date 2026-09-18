# Optional ingredients and cookware

* Proposal: [0018-optional-components](0018-optional-components.md)
* Authors: [Alexey Dubovskoy](https://github.com/dubadub)
* Status: **Awaiting review**

## Introduction

This proposal moves the optional marker `?` from a parser extension into the Cooklang specification. Writing `@?thyme` or `#?stand mixer{}` marks that ingredient or cookware as optional: the recipe works without it. The syntax is already implemented in [cooklang-rs](https://github.com/cooklang/cooklang-rs/blob/main/extensions.md#modifiers) as part of the "component modifiers" extension; this proposal standardises the `?` modifier only, and defines what applications should do with it. It also extends the [shopping list format](0016-shopping-list-format.md) so a list can record which optional ingredients the user decided to buy.

Discussion thread: [Support optional ingredients](https://github.com/cooklang/spec/discussions/50)

## Motivation

Recipes are full of optional components: a garnish, a pinch of chilli for those who like heat, a thermometer that helps but isn't required. Today the specification has no way to say so. Authors fall back on prose:

```cooklang
Garnish with @fresh thyme{2%sprigs} (optional).
```

This reads fine to a human but tools can't see it. Thyme lands on the shopping list as a required item, and a "what can I cook with what I have" search rejects the recipe because thyme is missing from the pantry.

The cooklang-rs parser has supported `@?thyme` for a long time behind the `COMPONENT_MODIFIERS` extension flag, and authors have adopted it. Because it is an extension and not part of the spec, support is inconsistent:

- Tools that enable the extension (e.g. cooklang-chef, the cookbook generator) treat `@?thyme` as optional thyme.
- Tools that follow the spec strictly (e.g. CookCLI) parse the same text as a required ingredient literally named `?thyme`.

The same recipe therefore looks correct in one tool and corrupted in another. [cookcli#502](https://github.com/cooklang/cookcli/pull/502) works around this by enabling the whole modifiers extension in CookCLI, but that flag bundles four other modifiers (`@` recipe, `&` reference, `-` hidden, `+` new) that are not in the spec either, and enabling it changes the meaning of existing recipes as a side effect (e.g. `@-salt`). The clean fix is for the spec to define optional components, so every implementation can support `?` unconditionally.

## Proposed solution

A `?` placed immediately after the `@` or `#` sigil marks that ingredient or cookware as optional:

```cooklang
---
servings: 2
---

Fry @eggs{2} in @butter{10%g}. Use a #?splatter guard{} if you have one.

Season with @salt and @?chilli flakes{1%pinch}.

Serve on @toast{2%slices}, garnished with @?chives.
```

The parsed recipe has:

| Ingredient | Quantity | Optional |
| --- | --- | --- |
| eggs | 2 | no |
| butter | 10 g | no |
| salt | | no |
| chilli flakes | 1 pinch | **yes** |
| toast | 2 slices | no |
| chives | | **yes** |

and cookware `splatter guard` marked optional.

The `?` is not part of the name. It works with every existing form of ingredient and cookware: single word (`@?chives`), multi-word (`@?chilli flakes{}`), with quantity (`@?chilli flakes{1%pinch}`), and recipe references (`@?./sauces/chimichurri{}`).

Compared with the "(optional)" prose workaround, the marker is shorter, is language independent, and lets applications treat optional components differently in ingredient lists, shopping lists and pantry matching.

## Detailed design

### Grammar

The `ingredient` and `cookware` productions gain an optional marker directly after the sigil:

```ebnf
ingredient           = one word ingredient | multiword ingredient ;
one word ingredient  = "@", [ optional marker ], one word component ;
multiword ingredient = "@", [ optional marker ], multiword component ;

cookware             = one word cookware | multiword cookware ;
one word cookware    = "#", [ optional marker ], one word component ;
multiword cookware   = "#", [ optional marker ], multiword component ;

optional marker      = "?" ;
```

Rules:

1. The marker must follow the sigil immediately. `@? thyme` is not an optional ingredient; per existing rules a sigil followed by whitespace is plain text.
2. At most one marker is recognised. In `@??thyme` the first `?` is the marker and the name is `?thyme`.
3. Everything after the marker follows the existing naming rules unchanged. A `?` anywhere else is ordinary text: `Add @salt{}?` is a required ingredient followed by a question mark.
4. Timers are unaffected. `~?{5%minutes}` has no special meaning.

### Parsed model

Parsers must expose a boolean `optional` property on every ingredient and cookware item, defaulting to `false`. The name must not include the marker.

In the canonical test format an optional ingredient looks like:

```yaml
testOptionalIngredient:
  source: |
    Garnish with @?fresh thyme{2%sprigs}
  result:
    steps:
      -
        - type: text
          value: "Garnish with "
        - type: ingredient
          name: "fresh thyme"
          quantity: 2
          units: "sprigs"
          optional: true
    metadata: {}
```

Existing tests are unaffected because `optional` defaults to `false` and may be omitted when false.

### Optionality is per occurrence

The flag belongs to one occurrence of a component in the text, not to the ingredient as a whole. The same ingredient can be required in one place and optional in another:

```cooklang
Stir @parmesan{100%g} into the sauce.

Top with extra @?parmesan{50%g} before serving.
```

### Aggregation

When building a recipe's ingredient list, required and optional occurrences of the same ingredient are aggregated **separately**. The example above produces two entries:

- parmesan, 100 g
- parmesan, 50 g (optional)

rather than one entry of 150 g. This keeps quantities truthful: the required total never includes amounts the cook may skip, and someone who decides against every optional item can still read off exactly what they need.

Within each group, the usual aggregation rules apply (quantities with compatible units are summed). The same applies to cookware.

### Scaling

Optional components scale exactly like required ones.

### Recipe references

An optional recipe reference, `@?./sauces/chimichurri{}`, marks the whole referenced recipe as optional. When an application expands the reference into its ingredients (e.g. for a shopping list), all of those ingredients are treated as optional in the context of the referencing recipe, regardless of how they are marked in the referenced recipe itself.

### Shopping list format

Optional ingredients are opt-in on a shopping list. This proposal extends the [shopping list format](0016-shopping-list-format.md) with a **selection line** that records each optional ingredient the user accepted, together with the amount to buy:

```
./Breakfast/Eggs on toast{2}
  ? chilli flakes{2%pinch}
  ./Components/Salsa{2}
./Salads/Boring{2}
olive oil{4%l}
```

Taking Eggs on toast to be the recipe from the [example above](#proposed-solution), this reads as: "Eggs on toast ×2, including 2 pinches of its optional chilli flakes, with Salsa ×2; Boring salad ×2; plus 4 l of olive oil." The optional chives of Eggs on toast, and any optional ingredients of Salsa and Boring salad, are not on the list.

New line forms in the `.shopping-list` grammar:

| Line form | Meaning |
|---|---|
| `? name{quantity%unit}` | Accepted optional ingredient of the parent recipe, with the amount to buy (indented only) |
| `? name{quantity}` | Same, quantity without unit |
| `? name` | Same, no quantity |
| `? ./path{multiplier}` | Accepted optional recipe reference of the parent recipe (indented only) |

```
line           = blank | comment | top_level_item | indented_ref | selection
selection      = INDENT "?" " " (ingredient | recipe_ref) NL
```

`ingredient` and `recipe_ref` are the existing productions of proposal 0016.

Rules:

1. A recipe reference on its own contributes only the **required** ingredients of that recipe. Each selection line beneath it adds one optional ingredient, which the list must show marked as optional.
2. A selection line belongs to the recipe reference one indentation level above it, following the existing indentation rules. It may appear under a reference at any nesting level. The prefix is exactly `?` followed by a single space, mirroring the check file's `+ name` / `- name` entries.
3. The quantity is the **final amount to buy**. The application writing the line computes it from the recipe — the aggregated optional amount of that ingredient (see [Aggregation](#aggregation)) with the reference's scaling already applied — and readers use it exactly as written, like a free-hand item. Readers must not multiply it by the parent's multiplier. In the example the recipe calls for 1 pinch and the reference is `{2}`, so the line says `{2%pinch}`. Braces are omitted when the ingredient has no quantity.
4. A selection line is self-contained: totals for optional ingredients are computed from the selection lines alone, without looking the ingredient up in the recipe. Required amounts of the same ingredient still come from expanding the recipe reference, and stay separate from the optional amount.
5. A recipe may use the same optional ingredient more than once — `@?parmesan{50%g}` on top and `@?parmesan{20%g}` to garnish. Because the line carries the amount, the user can accept all of those occurrences or only some of them. Several selection lines with the same name under one reference are allowed, and **each contributes its amount**; readers sum them subject to the usual unit compatibility. A writer may merge accepted occurrences with compatible units into one line or write one line per occurrence — the result is the same:

   ```
   ./Mains/Risotto
     ? parmesan{70%g}          -- both occurrences accepted, merged
   ```

   ```
   ./Mains/Risotto
     ? parmesan{50%g}          -- only the topping accepted
   ```

   When accepted occurrences cannot be summed into one quantity (e.g. `1%pinch` and `5%g`), the writer emits one selection line per quantity.
6. Because the quantity is a snapshot, an application that changes a reference's multiplier must rewrite the selection lines beneath it. Applications that have the recipe available may refresh selection lines whose recipe has changed, and may drop lines whose name no longer matches an optional ingredient of the parent recipe. A line that cannot be mapped back to specific occurrences unambiguously (for example after a partial acceptance) should be left as written. Names are compared case-insensitively with leading and trailing whitespace trimmed, as for the check file. Someone editing a multiplier by hand is responsible for adjusting the selection lines too.
7. An optional recipe reference such as `@?./sauces/chimichurri{}` is accepted with its path as written in the recipe: `? ./sauces/chimichurri{2}`. The braces hold its multiplier, and the line otherwise behaves exactly like a nested recipe reference: it is expanded into its required ingredients, and selection lines for its own optional ingredients may be indented beneath it. Ingredients it contributes are shown as optional.
8. Selection lines are only recognised when indented. Top-level free-hand items were added explicitly by the user and have no notion of optionality.
9. Serializers write selection lines directly after their parent reference, before any nested recipe references, with 2 spaces of indentation per level, and format quantities as for free-hand ingredients.

The check file is unchanged. Its entries are global by name, so `+ parmesan` checks off both the required and the optional parmesan entries.

Indented lines that are not recipe references were previously invalid, so no valid existing `.shopping-list` file changes its parse. Parsers implementing only proposal 0016 will reject or ignore selection lines; ignoring them degrades gracefully to a list of required ingredients.

### Scope

This proposal standardises the `?` marker only. The other cooklang-rs component modifiers (`&` reference, `-` hidden, `+` new, `@` recipe) remain extensions. They keep working in parsers that enable them, including in combination with `?`, but that behaviour is outside the specification.

### Backward compatibility

Under the current specification `@?thyme` is a required ingredient named `?thyme`. After this change it is an optional ingredient named `thyme`. Ingredient names that intentionally start with `?` are not known to exist in practice, while recipes already using `@?` with the intended optional meaning are common, so this change fixes far more recipes than it could break. No migration is needed.

## Effect on applications which use Cooklang

### Specification updates

Once accepted, the `?` marker is documented in the language specification (README and EBNF) with canonical tests, and the selection line is added to the shopping list section of the conventions.

### Display

Applications should visibly distinguish optional components wherever components are shown:

1. **Ingredient and cookware lists** — mark optional entries, e.g. with an "optional" label or badge. The label should be localised.
2. **Inline in steps** — the step text usually already conveys it ("garnish with chives if you like"), so inline marking is recommended but may be subtle.

### Shopping lists

1. When a recipe with optional ingredients is added to a shopping list, applications should ask the user which of them to include, and record each accepted one as a [selection line](#shopping-list-format) with its scaled amount. When an optional ingredient is used more than once, the choice may be offered per ingredient (all occurrences together) or per occurrence. Applications that cannot ask add the required ingredients only. When the user changes a recipe's multiplier, the application rewrites the amounts on its selection lines.
2. Optional ingredients that are on the shopping list must be marked as optional, so the shopper knows they can be skipped if unavailable.
3. Applications should make optional ingredients that were not accepted discoverable — for example as suggestions under the recipe — so they can be added later and are not simply forgotten.
4. Following the aggregation rule, optional and required amounts of the same ingredient stay distinguishable within a recipe. When combining across recipes, applications should still keep optional amounts distinguishable from required ones.

### Pantry matching

Features that match recipes against available ingredients ("what can I cook?") should not count a missing optional ingredient as missing. A recipe whose only absent ingredients are optional is a full match.

### CookCLI (terminal and web-server)

CookCLI should:

1. Parse the `?` marker unconditionally, without enabling the rest of the component modifiers extension.
2. Mark optional ingredients and cookware in all recipe outputs: human, markdown, and the web UI (recipe view and cooking mode), and expose an `optional` field in JSON and YAML output.
3. In `shopping-list`, leave optional ingredients out by default and add a flag to include them (e.g. `--include-optional`), marking them in the output. Read and write selection lines in `.shopping-list` files. In the web UI, offer the choice when adding a recipe to the list.
4. In pantry-based recipe matching, ignore missing optional ingredients.

No new sub-command is required. Much of the display work is already done in [cookcli#502](https://github.com/cooklang/cookcli/pull/502).

### Mobile applications

Mobile apps should:

1. Show an "optional" label next to optional entries in the recipe's ingredient and cookware lists, and in cooking mode.
2. When adding a recipe to the shopping list, present its optional ingredients unticked for the user to accept, and write a selection line for each accepted one.
3. Mark optional items in the shopping list view, and let the user add or remove optional ingredients of a recipe already on the list.

### Parsers

Parsers that implement this as an extension today (cooklang-rs) should make the `?` marker part of the core syntax, available with all extensions disabled. Other parsers (cooklang-ts, cooklang-swift, cooklang-c, etc.) need the grammar change and the new `optional` property.

## Alternatives considered

### Suffix marker: `@thyme{}?`

Reads naturally, but is ambiguous with ordinary punctuation — `Did you add the @salt{}?` is a legitimate sentence. It is also incompatible with the recipes that already use the prefix form.

### Marker inside the braces: `@thyme{?}` or `@thyme{2%sprigs?}`

This overloads the quantity slot, conflicts with free-text quantities, and forces braces onto single-word ingredients. It is also incompatible with existing recipes.

### Standardise all component modifiers at once

Promoting the whole cooklang-rs modifiers extension would let implementations simply flip the existing flag. It was rejected because the modifiers are independent features with very different levels of maturity: `@@recipe` overlaps with the `@./path` recipe reference syntax already in the spec, and `&`/`+` are tied to the modes extension. Each deserves its own discussion. `?` is the most widely used, has the simplest semantics, and is the one causing visible inconsistencies between tools today.

### Merge optional and required occurrences into one list entry

An ingredient list could show a single `parmesan 150 g` entry, marked optional only if every occurrence is optional. This is simpler to implement, but it overstates the required amount and hides the fact that part of it can be skipped. Separate entries keep the numbers honest.

### Include optional ingredients by default, and record exclusions

The shopping list could include every optional ingredient unless the user declined it, recording declined items under the recipe (e.g. `- chives`). This never silently leaves anything out, but the file then no longer says what is being bought: the contents depend on whatever optional ingredients the recipe has at the moment the list is expanded, and adding an optional ingredient to a recipe later silently grows existing lists. Opt-in selection keeps the list explicit and lean, and matches what "optional" means. The cost — a cook forgetting a garnish they would have wanted — is addressed by asking when the recipe is added and by keeping unaccepted items discoverable.

### A per-recipe switch instead of per-ingredient selection

A single marker on the recipe reference ("with optional ingredients" / "without") is simpler to parse and needs only one yes/no prompt, but it is all-or-nothing: the user cannot take the chives and skip the chilli.

### Record the choice outside the shopping list file

The selection could be kept in application state or in a separate file next to `.shopping-checked`. That would leave the 0016 format untouched, but the list would no longer be self-contained: sharing or syncing `.shopping-list` would lose the user's choices.

### Selection lines without a quantity, or with an unscaled quantity

A selection line could carry only the name (`? chives`), with the amount derived from the recipe whenever the list is expanded, or carry the recipe's base amount for readers to multiply. Both keep the amount in step with the multiplier automatically, but they make every reader do more work: look up and aggregate the optional occurrences in the recipe, or apply scaling — which a plain multiplication gets wrong for fixed and text quantities. A name-only line also cannot express accepting just one of several optional uses of the same ingredient. Storing the final amount makes totals a simple sum, at the cost of rewriting the lines when the multiplier changes.

### Optional steps and sections

[Discussion #122](https://github.com/cooklang/spec/discussions/122) proposes extending optionality to whole steps and sections. That is a larger change with its own syntax questions and is left for a future proposal. Nothing here prevents it.

## Acknowledgments

Thanks to [Zheoni](https://github.com/Zheoni) for designing and implementing the modifiers extension in cooklang-chef and cooklang-rs, where this syntax originated; to [Heather Kemp](https://github.com/hekemp) for [cookcli#502](https://github.com/cooklang/cookcli/pull/502), which surfaced the inconsistency between tools and prototyped the UI; and to everyone who took part in discussions [#50](https://github.com/cooklang/spec/discussions/50) and [#122](https://github.com/cooklang/spec/discussions/122).
