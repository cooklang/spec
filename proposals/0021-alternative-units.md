# Alternative units and dual quantities

* Proposal: [0021-alternative-units](0021-alternative-units.md)
* Authors: [Alexey Dubovskoy](https://github.com/dubadub)
* Review Manager: TBD
* Status: **Awaiting review**
* Related: [unit system metadata (PR #148)](https://github.com/cooklang/spec/pull/148), [0011 ranges](https://github.com/cooklang/spec/pull/109), [0017 yield](0017-yield-metadata.md)

## Introduction

This proposal lets an author write the same amount of an ingredient in more than one unit, so one recipe file can be published in metric and imperial (or US customary) at the same time:

```cooklang
Whisk @flour{2%cups|260%g} with @butter{1/2%cup|115%g}.
```

The amounts inside the braces are separated by `|`. The first one is the **primary** amount, as the author measured it; the others are **equivalents** supplied by the author. All of them describe one quantity of one ingredient, all of them scale, and an application shows whichever the reader prefers.

It also extends the `unit system` metadata key proposed in [PR #148](https://github.com/cooklang/spec/pull/148) so a list of systems names what each position means:

```yaml
---
unit system: [us customary, metric]
---
```

Discussion threads: [Alternative unit and range quantity](https://github.com/cooklang/spec/discussions/84), [Support for Unit and Serving Size Conversions](https://github.com/cooklang/spec/discussions/142), [Keyword for units for internationalization](https://github.com/cooklang/spec/discussions/51), [cookcli#432](https://github.com/cooklang/cookcli/issues/432) (fraction and unit display).

## Motivation

Good recipes are often published with two measurements:

- Baking recipes give volume and weight: "2 cups (260 g) flour". Weight is what makes the recipe reliable; volume is there for readers without a scale. In discussion #84 this is called "my biggest issue with cooklang so far".
- UK and US publishers, and anyone writing for both audiences, give metric and imperial side by side: "1 lb (450 g) minced beef", "350 °F (180 °C)".
- Cookbooks being digitised already have both, and the author wants to keep both rather than choose one.

Cooklang can hold one amount per ingredient, so today authors do one of three things, each of them bad:

```cooklang
@flour{2%cups}(260 g)          -- second amount hidden in the note: does not scale, not machine-readable
@flour{260%g} (2 cups)         -- plain text after the ingredient: does not scale, breaks the step on doubling
@flour{2%cups} @flour{260%g}   -- two ingredients: the list and shopping list ask for both
```

Doubling the recipe in the first two cases gives "4 cups (260 g)", which is wrong. The cook.md importer, the AI editor and CookCLI's import all meet "2 cups (260 g)" on every second recipe site and currently have to drop one amount or squash it into the note.

Client-side conversion does not solve this on its own. Converting 2 cups of flour into grams needs the flour's density, and a client does not know that 2 cups of *this* author's spooned-and-levelled flour is 260 g and not 240 g or 280 g. The author knows. [PR #148](https://github.com/cooklang/spec/pull/148) rightly keeps density tables out of the language and gives clients a `unit system` hint for same-dimension conversion. That covers `500%g` → ounces. It cannot cover cups → grams, and it cannot carry an author's own rounded equivalents ("1 lb (450 g)", not "1 lb (453.59 g)"). The two proposals are complementary: one tells clients how to convert, the other lets authors say what the conversion is when they know better.

## Proposed solution

Inside an ingredient's braces, `|` separates equivalent amounts:

```cooklang
---
title: Brown butter chocolate chip cookies
servings: 24
unit system: [us customary, metric]
---

Melt @butter{1%cup|225%g} in a #saucepan{} and cook until it smells nutty, ~{5%minutes}.

Whisk in @brown sugar{1%cup|200%g}, @caster sugar{1/2%cup|100%g}, @eggs{2} and @vanilla extract{2%tsp|10%ml}.

Fold in @plain flour{2 1/4%cups|280%g}, @baking soda{1%tsp|5%g} and @salt{1/2%tsp|3%g}, then @dark chocolate{8%oz|225%g}(chopped).

Bake at 350 °F (180 °C) for ~{10-12%minutes}.
```

The parsed recipe has:

| Ingredient | Primary | Equivalents |
| --- | --- | --- |
| butter | 1 cup | 225 g |
| brown sugar | 1 cup | 200 g |
| caster sugar | 1/2 cup | 100 g |
| eggs | 2 | |
| vanilla extract | 2 tsp | 10 ml |
| plain flour | 2 1/4 cups | 280 g |
| baking soda | 1 tsp | 5 g |
| salt | 1/2 tsp | 3 g |
| dark chocolate | 8 oz | 225 g |

The metadata says position 1 is US customary and position 2 is metric. A reader who prefers metric sees "225 g butter"; a US reader sees "1 cup butter"; an app may also show both, "1 cup (225 g) butter". Doubling the recipe gives "2 cups (450 g) butter" everywhere. Eggs have one amount, which is right for every reader.

The ingredient list, shopping list and pantry treat each line as **one** ingredient with one amount in the reader's chosen system. Nothing is listed twice.

## Detailed design

### Grammar

Ingredients get their own component productions, so the change does not reach cookware or timers, which keep the existing `amount`:

```ebnf
one word ingredient  = "@", [ optional marker ], one word ingredient component ;
multiword ingredient = "@", [ optional marker ], multiword ingredient component ;

one word ingredient component  = word, [ "{", [ ingredient amount ], "}" ] ;
multiword ingredient component = word, { text item - "{" }-, "{", [ ingredient amount ], "}" ;

ingredient amount    = alternative amount, { { white space }, "|", { white space }, alternative amount } ;
alternative amount   = alternative quantity | ( alternative quantity, "%", alternative units ) ;
alternative quantity = { text item - "%" - "|" - "}" }- ;
alternative units    = { text item - "|" - "}" }- ;
```

If proposal 0020 is accepted, `[ optional marker ]` becomes `[ markers ]` as defined there. When proposal 0011 is accepted, `alternative quantity` also takes the `range | number` forms defined there, and each alternative may be a range: `@chicken wings{1 1/2-2%lb|700-900%g}`.

Rules:

1. Alternatives are separated by `|`. White space around `|` is ignored: `{2%cups | 260%g}` is the same as `{2%cups|260%g}`.
2. Each alternative is a complete amount with its own quantity and optional unit. `@eggs{2|100%g}` is valid (2 eggs, or 100 g of egg).
3. The first alternative is the primary. Order is meaningful and parsers must preserve it.
4. An empty alternative is an error: `{2%cups|}` and `{|260%g}` are reported, and the empty part is dropped.
5. Alternatives apply to ingredients only. Cookware and timer amounts are unchanged, and a `|` inside them stays part of the text. (`~{1%hour|60%minutes}` has no use, and cookware quantities have no units to alternate.)
6. `|` between components, as in `@butter{}|ghee{}` for substitute ingredients, and `|` in a name, as in the cooklang-rs alias extension `@white wine|wine{}`, are outside the braces and unaffected. This proposal claims `|` only inside an ingredient's braces.

### Fixed quantities

The `=` prefix from [Fixed Quantities](../conventions.md#fixed-quantities) goes before the first alternative and fixes the whole amount: `@salt{=1%tsp|5%g}` stays at 1 tsp and 5 g at any scale. Fixing one alternative but not another would make them disagree, so `=` in front of a later alternative is a warning and is ignored.

### Parsed model

Parsers keep the existing `quantity` and `units` of an ingredient as the **primary** amount, and add an `alternatives` list holding the other amounts in order. A consumer that knows nothing about this proposal keeps working and sees the author's primary measurement.

```yaml
testAlternativeUnits:
  source: |
    Add @flour{2%cups|260%g}
  result:
    steps:
      -
        - type: text
          value: "Add "
        - type: ingredient
          name: "flour"
          quantity: 2
          units: "cups"
          alternatives:
            - quantity: 260
              units: "g"
    metadata: {}
```

`alternatives` defaults to an empty list and may be omitted from tests when empty. Canonical tests to add with the docs change: two alternatives, three alternatives, an alternative without units (`{2|100%g}`), spaces around `|`, a fraction and a mixed number in one alternative, a text quantity in one alternative (`{a handful|30%g}`), a fixed amount (`{=1%tsp|5%g}`), an empty alternative (error), `|` in a cookware amount (text, unchanged), and a range in each alternative once 0011 lands.

### Naming the unit systems

[PR #148](https://github.com/cooklang/spec/pull/148) proposes an optional `unit system` key with the values `metric`, `imperial` and `us customary`, describing the system the recipe is written in. This proposal keeps that meaning for a scalar value and lets the key also hold a list:

```yaml
unit system: metric                      # single-system recipe (as in PR #148)
unit system: [us customary, metric]      # alternative 1 is US customary, alternative 2 is metric
unit system: [metric, imperial]          # a UK book: metric first, imperial second
```

With a list, entry *n* names the system of the *n*th alternative of every ingredient amount in the recipe. This is what an application shows on its toggle ("US / Metric"), and it settles the units that are spelled the same in several systems: `cup` and `pint` in a recipe whose second position is `imperial` are Imperial cups and pints, and `tsp` in a `metric` position is a 5 ml metric teaspoon.

Rules:

1. Values are the canonical ones from PR #148, lower case. Other strings are allowed as labels (`[volume, weight]`, `[cups, grams]`) and may be shown on a toggle, but carry no conversion meaning.
2. An amount with fewer alternatives than the list, such as `@eggs{2}`, is the same amount in every system. This is the normal case for counts, pinches and "to taste".
3. An amount with more alternatives than the list has unnamed extra positions. They are kept, displayed when the reader shows all alternatives, and never selected by system.
4. Without the key, or with a scalar value, applications may still infer an alternative's system from its unit (`g` is metric, `oz` is not). A declared list takes precedence over inference.
5. A list must not repeat a value. `[metric, metric]` is a warning and only the first position is selectable by system.

The `yield` key already takes a list (`[500%ml, 2%cups]`). It is unaffected: its entries are matched by unit, as [0017](0017-yield-metadata.md) defines, not by position.

### Selecting an alternative

An application that shows one amount per ingredient picks it like this:

1. If the reader has chosen a system and the recipe's `unit system` list names it, use that position. If the ingredient has no alternative at that position, use the primary.
2. Otherwise, if the reader has a preferred system, use the first alternative whose unit belongs to it.
3. Otherwise use the primary.

An application may convert the selected amount further (for example `1000 g` to `1 kg`, or `8 tbsp` to `1/2 cup` after scaling), following PR #148's client-owned conversion rules. An author-supplied alternative always takes precedence over a computed conversion: if the reader wants grams and the recipe gives 260 g, show 260 g, not the client's own conversion of 2 cups.

An application may also show all alternatives at once. The conventional form is primary first, then the rest in parentheses: "2 cups (260 g)".

### Scaling

Every alternative scales by the same factor as the primary. `{2%cups|260%g}` at double is `{4%cups|520%g}`. Text quantities in an alternative do not scale, as today. Fixed amounts (`=`) do not scale in any alternative.

Applications should apply their usual rounding and fraction display to each alternative separately, in a way suited to its unit: imperial and US amounts usually as fractions, metric as decimals. The values must not be recomputed from each other after scaling; each is the author's own value multiplied by the factor.

### Aggregation

When building the ingredient list, the shopping list or a pantry check, applications work in one system at a time. For each occurrence they select an alternative as above and sum the selected amounts by the existing rules. Two occurrences of `butter` written as `{1%cup|225%g}` and `{2%tbsp|30%g}` are listed as "1 cup 2 tbsp" (or "1 1/8 cup" after conversion) for a US reader and "255 g" for a metric reader.

If the occurrences of one ingredient do not all have an alternative in the chosen system, the application falls back to the primary amounts for that ingredient rather than mixing systems silently. It may then convert by its own rules, or list the amounts separately as today.

### Recipe references

A reference with alternatives, `@./sauces/hollandaise{150%ml|2/3%cup}`, scales the referenced recipe using the first alternative that matches an entry in its `yield` (proposal 0017). If none matches, it is an error, as with a single amount.

### Shopping lists and pantry

The [shopping list format](../conventions.md#shopping-list-format) stores recipe references and expands them when read, so it needs no change: a reader that selects alternatives as above produces a list in the reader's system. Lines that a writer records with an explicit amount, such as accepted optional ingredients, are written in the system the user was viewing.

Pantry matching compares a recipe amount to a pantry entry using whichever alternative shares a unit, or a convertible unit, with the pantry entry. A pantry that tracks flour in grams can match `{2%cups|260%g}` exactly, without a density.

### Scope

This proposal standardises:

- the `|` alternative syntax inside ingredient braces;
- primary-first order, scaling and aggregation of alternatives;
- the list form of the `unit system` key.

It does not standardise conversion ratios, rounding, density data or which units belong to which system. Those stay client-owned, as PR #148 proposes. It also does not cover temperatures and other numbers written in step text ("350 °F (180 °C)"), which are left to the temperature proposal (0009).

### Backward compatibility

Under the current specification `@flour{2%cups|260%g}` is an ingredient with quantity text `2` and unit `cups|260%g` (the unit runs to the closing brace). That is not a useful parse and no recipe relies on it. After this change it is 2 cups with a 260 g equivalent.

A `|` inside a single text quantity, such as `@cheese{cheddar|gouda}`, would now be read as two text alternatives. Such recipes are not known to exist; the form that people do write for substitutes puts the `|` between components (`@cheddar{}|gouda{}`), which is unaffected. Parsers that read the new `alternatives` field from such a recipe still get the first part as the quantity.

`unit system` with a scalar value keeps the meaning PR #148 gives it. The list form is new.

## Effect on applications which use Cooklang

### Specification updates

Once accepted, the README gains a section on alternative amounts under ingredients, the EBNF gets the `ingredient amount` production, canonical tests get the `alternatives` property and the tests listed above, and the conventions get the list form of `unit system` (in Canonical metadata, alongside PR #148's scalar form) and the selection, aggregation and pantry rules.

### Display

1. **Inline in steps.** Show the selected alternative, or all of them as "2 cups (260 g)". A tap or hover on the amount may reveal the others.
2. **Unit system toggle.** When `unit system` is a list, show a toggle labelled with its values, defaulting to the reader's preference, otherwise to the first position. The toggle should be remembered per reader, not per recipe.
3. **Ingredient list.** One line per ingredient, in the selected system. The "show both" setting applies here too.

### Shopping lists

Built in the system the user chose. The same ingredient from two recipes in different systems is added up only if both have an amount in the chosen system, or can be converted by the client.

### Pantry

Matched on any alternative, as described above.

### CookCLI (terminal and web-server)

CookCLI should:

1. Parse alternatives and expose `alternatives` in JSON and YAML output.
2. Add a `--units <system>` option to `recipe` and `shopping-list` (and a matching setting in the web UI) that selects alternatives by the rules above, and a way to show all alternatives in human and markdown output.
3. Scale every alternative with `--scale` and servings scaling.
4. In `doctor`, warn on an empty alternative, a misplaced `=`, a `unit system` list with repeated values, and, optionally, on two alternatives in the same dimension that disagree by more than a few percent (`{1%cup|300%ml}`), which is usually a typo.
5. In the `cooklang-format` crate, keep alternatives and their order when reformatting.

### Mobile applications

Mobile apps should add the unit system toggle to the recipe view and cooking mode, use the selected system in the ingredient tab and when adding to the shopping list, and scale all alternatives.

### Editors and importers

Importers (cook.md, the AI editor, `cook import`) should turn "2 cups (260 g) flour" into `@flour{2%cups|260%g}` instead of dropping one amount or moving it into the note, and should set `unit system` when the source labels its columns ("US / Metric" toggles are common on recipe sites). The AI editor can offer "add metric equivalents" as an explicit, reviewable action, so the author checks densities rather than readers trusting a computed value. Syntax highlighters should treat `|` inside ingredient braces as a separator.

### Parsers

cooklang-rs, cooklang-ts, cooklang-swift, cooklang-c and the others need the grammar change and the `alternatives` property. The change is local to amount parsing inside ingredient braces.

## Alternatives considered

### Client-side conversion only

This is PR #148's approach, and it is the right default for same-dimension conversion. It cannot convert volume to weight without densities, and it cannot reproduce an author's rounded figures ("1 lb (450 g)"). Recipes that already carry both amounts would lose one. This proposal builds on PR #148 rather than replacing it.

### Comma separator: `@water{1%kg, 1.25%cup}`

Proposed in #84. Commas already appear in text quantities and in the wild as preparation notes inside braces (`{1, finely chopped}`), so a comma is ambiguous. `|` has no current use inside braces.

### Double pipe: `{2.5%cups||370%g}`

Suggested in #84 to read as a logical "or". It is more typing for no gain in clarity, and a single `|` already reads as "or" in the substitute-ingredient form `@butter{}|ghee{}`.

### A second set of braces or parentheses: `@flour{2%cups}{260%g}`, `@flour{2%cups}(260%g)`

A second brace pair is new syntax with no precedent and is easy to mistake for a timer or cookware amount. Parentheses are already the preparation note; overloading them breaks every recipe that writes "(sifted)".

### Inline system tags: `{2%cups@us|260%g@metric}`

Tagging each alternative makes every amount longer and repeats the same tag on every line of a recipe. In practice a recipe uses the same systems in the same order throughout, which is exactly what a positional metadata list expresses once. Inference from units still covers recipes without the key.

### A metadata map from units to systems

```yaml
unit systems:
  cup: us customary
  g: metric
```

This handles ambiguous unit names but not the toggle: it does not say which positions belong together, and an amount like `{1%cup|240%ml}` in a recipe whose author meant both to be US customary cannot be described. The positional list is shorter and answers both questions.

### Two ingredients

`@flour{2%cups}` and `@flour{260%g}` duplicate the ingredient in lists and on shopping lists and require hiding one of them, which loses the link between the two amounts.

### Separate files per system

A metric and an imperial copy of every recipe drift apart on the first edit, and references (`@./sauce{}`) have to pick one.

## Acknowledgments

Thanks to [ziofat](https://github.com/ziofat) for starting discussion [#84](https://github.com/cooklang/spec/discussions/84), [adsilcott](https://github.com/adsilcott) for the baking case, [jphastings](https://github.com/jphastings) for suggesting `|` inside braces, [Zie0](https://github.com/Zie0) and [shicks](https://github.com/shicks) for the syntax and "1 15-oz can" cases there, [RPGillespie6](https://github.com/RPGillespie6), [stevenbell](https://github.com/stevenbell) and the others in [#142](https://github.com/cooklang/spec/discussions/142), [umgefahren](https://github.com/umgefahren) for [#51](https://github.com/cooklang/spec/discussions/51), and [Luka Hummel](https://github.com/LukaHummel) for the `unit system` metadata proposal in [PR #148](https://github.com/cooklang/spec/pull/148) that this one extends.
