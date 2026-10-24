# Range quantities

* Proposal: [0011-ranges](0011-ranges.md)
* Authors: [Alexey Dubovskoy](https://github.com/dubadub)
* Status: **Awaiting review**

## Introduction

This proposal moves range quantities from a parser extension into the Cooklang specification. Writing `@garlic{2-3%cloves}` or `~{10-15%minutes}` gives an ingredient, cookware item or timer a numeric range instead of a single number. The syntax is already implemented in [cooklang-rs](https://github.com/cooklang/cooklang-rs/blob/main/extensions.md#range-values) behind the `RANGE_VALUES` extension flag; this proposal standardises it and defines how ranges scale, add up, convert and appear in shopping lists.

Discussion threads: [Alternative unit and range quantity](https://github.com/cooklang/spec/discussions/84), [Representing quantities without an associated ingredient](https://github.com/cooklang/spec/discussions/96).

## Motivation

Recipes are rarely exact. "2–3 cloves of garlic", "add 1–2 tablespoons of water until the dough comes together", "bake for 25–30 minutes" are how cooks actually write. Today the specification offers no way to say this. Authors either pick one number and lose the information, or write the range in prose:

```cooklang
Add @water{1%tbsp} (1-2 tbsp, as needed) and bake for ~{25%minutes} (25-30 minutes).
```

Most authors do neither. They write `@water{1-2%tbsp}` and `~{25-30%minutes}` because that is the obvious thing to type, and because the first tool they used accepted it. Under the current specification both quantities are **text**: they display correctly, but they cannot be scaled, added to other quantities, or converted between units. The request has come up repeatedly since 2022 ([cookcli#49](https://github.com/cooklang/cookcli/issues/49), [discussion #84](https://github.com/cooklang/spec/discussions/84), [discussion #96](https://github.com/cooklang/spec/discussions/96)).

The cooklang-rs parser has supported ranges as numbers for years behind the `RANGE_VALUES` extension. Because it is an extension and not part of the spec, support is inconsistent across the ecosystem:

| Tool | Ranges parsed as | Consequence |
| --- | --- | --- |
| cooklang-chef, playground, `@cooklang/cooklang` npm package, language server | numbers | scale, add and convert correctly |
| CookCLI | text | ranges stay unchanged when a recipe is scaled ([cookcli#597](https://github.com/cooklang/cookcli/issues/597)) |
| iOS app (Swift bindings) | text | scaling fails on ranges; a workaround PR exists ([cooklang-rs#78](https://github.com/cooklang/cooklang-rs/pull/78)) |
| cook.md web | text | the AI importer is instructed to write `~{10-15%minutes}`, but the site that renders the result treats it as text |

The same recipe therefore scales in one tool and silently does not in another. Tools have started to paper over the gap: CookCLI's `doctor` command enables the extension privately just to validate timer units, and cook.md's import linter enables it to read the upper bound of a range. The clean fix is for the specification to define ranges, so every implementation parses them the same way.

## Proposed solution

A quantity written as two numbers joined by a hyphen is a **range**:

```cooklang
---
servings: 4
---

Crush @garlic{2-3%cloves} and fry in @olive oil{1-2%tbsp} for ~{2-3%minutes}.

Add @canned tomatoes{400%g}, @water{100-150%ml} and @chilli flakes{1/2-1%tsp}.

Simmer for ~{25-30%minutes} in a #saucepan{}.
```

The parsed recipe has:

| Ingredient | Quantity |
| --- | --- |
| garlic | 2–3 cloves |
| olive oil | 1–2 tbsp |
| canned tomatoes | 400 g |
| water | 100–150 ml |
| chilli flakes | ½–1 tsp |

and two timers, 2–3 minutes and 25–30 minutes.

Scaled to 8 servings the garlic becomes 4–6 cloves, the water 200–300 ml and the chilli flakes 1–2 tsp. Timers are not scaled, as today. Compared with writing the range in prose, the marked-up form is shorter, keeps the ingredient list honest, and lets applications scale, aggregate and convert the quantity like any other number.

## Detailed design

### Grammar

The `quantity` production gains a `range` alternative. The specification has so far left numbers implicit (the canonical tests define which quantities parse as numbers); this proposal writes them down so that `range` has something to refer to.

```ebnf
amount   = quantity | ( quantity, "%", units ) ;
quantity = range | number | { text item - "%" - "}" }- ;
range    = number, { white space }, "-", { white space }, number ;

number         = decimal | mixed number | fraction | integer ;
integer        = "0" | non-zero digit, { digit } ;
decimal        = [ integer ], ".", digit, { digit } ;
fraction       = integer, { white space }, "/", { white space }, integer ;
mixed number   = integer, { white space }-, fraction ;
digit          = "0" | non-zero digit ;
non-zero digit = "1" | "2" | "3" | "4" | "5" | "6" | "7" | "8" | "9" ;
```

Rules:

1. The alternatives of `quantity` are tried in the order written, against the whole quantity with surrounding whitespace trimmed. A quantity that matches `range` is a range even though it also matches the text alternative; otherwise it is a number if it matches `number`; otherwise it is text, exactly as today.
2. Both ends must be numbers. `{2-}`, `{-3}`, `{2-few}` and `{1-2 handfuls}` are text, as they are today. A negative number is not a number in this grammar, so `-` can only ever mean a range.
3. Exactly one hyphen separates the ends. `{1-2-3}` is text.
4. Whitespace around the hyphen is allowed: `{2 - 3}` and `{2-3}` are the same range. Whitespace inside a number follows the existing rules for fractions and mixed numbers, so `{1 1/2-2%cups}` is the range 1½–2.
5. The units after `%` apply to both ends. There is no way to give the two ends different units.
6. The separator is the ASCII hyphen-minus (U+002D). Typographic dashes such as `–` (U+2013) are text. Applications may render a range with an en dash or a localised word ("2 to 3"), but what is stored in the file is `-`.
7. The ends are kept exactly as written. Parsers must not reorder them, and `{3-2}` is a valid range with start 3 and end 2; a parser may warn about it. `{2-2}` is a range with equal ends; applications may display it as the single number 2, but it stays a range in the parsed model and is written back as `2-2`.
8. A fraction whose denominator is `0`, on its own or as either end of a range, is not a number. Parsers report an error, as cooklang-rs does today, and must not produce a numeric value for it.

The `number` productions codify how parsers already read numeric quantities, as fixed by the canonical tests (`1/2`, `1 / 2`, `1.5` and `250` are numbers; `01/2` is text). The only new production is `range`.

### Scope

This proposal standardises the `number-number` form inside `{}` only. Under its `ADVANCED_UNITS` extension cooklang-rs also accepts a unit without `%` after a number or a range, so that `{1-2 handfuls}` parses as the range 1–2 with the unit `handfuls`; that form stays an extension, and the canonical tests below assume all extensions are off.

### Parsed model

Parsers must expose a range as a numeric value with a `start` and an `end`, distinct from a text value, on ingredients, cookware and timers. Each end is a number of the same kind as a plain numeric quantity (so fractions stay fractions). Text quantities are unchanged.

In the canonical test format a range is written as a mapping with `start` and `end`:

```yaml
testIngredientRange:
  source: |
    @garlic{2-3%cloves}
  result:
    steps:
      -
        - type: ingredient
          name: "garlic"
          quantity:
            start: 2
            end: 3
          units: "cloves"
    metadata: {}

testIngredientRangeFraction:
  source: |
    @chilli flakes{1/2-1%tsp}
  result:
    steps:
      -
        - type: ingredient
          name: "chilli flakes"
          quantity:
            start: 0.5
            end: 1
          units: "tsp"
    metadata: {}

testIngredientRangeDecimal:
  source: |
    @cream{1.5-2.25%cups}
  result:
    steps:
      -
        - type: ingredient
          name: "cream"
          quantity:
            start: 1.5
            end: 2.25
          units: "cups"
    metadata: {}

testIngredientRangeMixedNumber:
  source: |
    @flour{1 1/2-2%cups}
  result:
    steps:
      -
        - type: ingredient
          name: "flour"
          quantity:
            start: 1.5
            end: 2
          units: "cups"
    metadata: {}

testIngredientRangeWithSpaces:
  source: |
    @eggs{2 - 3}
  result:
    steps:
      -
        - type: ingredient
          name: "eggs"
          quantity:
            start: 2
            end: 3
          units: ""
    metadata: {}

testTimerRange:
  source: |
    Bake for ~{25-30%minutes}
  result:
    steps:
      -
        - type: text
          value: "Bake for "
        - type: timer
          quantity:
            start: 25
            end: 30
          units: "minutes"
          name: ""
    metadata: {}

testRangeOpenEndedIsText:
  source: |
    @eggs{2-}
  result:
    steps:
      -
        - type: ingredient
          name: "eggs"
          quantity: "2-"
          units: ""
    metadata: {}

testRangeThreeNumbersIsText:
  source: |
    @eggs{1-2-3}
  result:
    steps:
      -
        - type: ingredient
          name: "eggs"
          quantity: "1-2-3"
          units: ""
    metadata: {}

testRangeEnDashIsText:
  source: |
    @eggs{2–3}
  result:
    steps:
      -
        - type: ingredient
          name: "eggs"
          quantity: "2–3"
          units: ""
    metadata: {}

testRangeLikeTextStaysText:
  source: |
    @water{1-2 handfuls}
  result:
    steps:
      -
        - type: ingredient
          name: "water"
          quantity: "1-2 handfuls"
          units: ""
    metadata: {}

testDecimalWithoutLeadingZero:
  source: |
    @milk{.5%cup}
  result:
    steps:
      -
        - type: ingredient
          name: "milk"
          quantity: 0.5
          units: "cup"
    metadata: {}
```

The last test pins down a case the written-down `number` grammar makes explicit: a decimal may omit the leading zero, as cooklang-rs already accepts. Existing tests are unaffected: every quantity that parses as a number or as text today still does, except the ones that match `range`, which no existing canonical test contains.

### Scaling

A range scales by multiplying both ends by the scaling factor: `@garlic{2-3%cloves}` doubled is 4–6 cloves. The fixed-quantity lock applies to the whole range, so `@salt{=1-2%tsp}` stays 1–2 tsp at any serving size. Timers are not scaled, as today.

Cookware quantities never carry units and do not scale under the [conventions](../conventions.md#what-doesnt-scale), so a cookware range such as `#ramekins{4-6}` is only displayed and aggregated.

### Aggregation

When quantities of the same ingredient are added (ingredient lists, shopping lists), ranges add end-to-end and a plain number adds to both ends:

| Occurrences | Total |
| --- | --- |
| `@garlic{2}` and `@garlic{1-2}` | 3–4 |
| `@water{100-150%ml}` and `@water{50-100%ml}` | 150–250 ml |
| `@flour{100%g}` and `@flour{1-2%tbsp}` | 100 g, 1–2 tbsp (incompatible units stay separate, as today) |

A range never collapses into a single number through aggregation, and a text quantity cannot be added to a range any more than it can be added to a number.

### Unit conversion

When an application converts a range to another unit or unit system, the target unit is chosen from the lower end alone, exactly as it would be for a single quantity equal to the lower end; the upper end is then converted to that same unit. `@water{100-150%ml}` converted to imperial becomes roughly 3.4–5.1 fl oz, not 100 ml–0.6 cups.

### Timers

A timer range such as `~{25-30%minutes}` is displayed as a range. When the application starts a countdown for it, the default duration is the **lower** end: a cook checks the oven at 25 minutes and keeps going if needed. Applications may let the user pick a different duration within the range, and should show the full range alongside the countdown.

### Shopping list format

The [shopping list format](0016-shopping-list-format.md) needs no grammar change: its `quantity` production already accepts any text, so a free-hand item `olive oil{1-2%tbsp}` and a selection line `? chilli flakes{1-2%pinch}` ([proposal 0018](0018-optional-components.md)) are already well-formed. This proposal adds meaning:

1. A quantity on a free-hand item or a selection line that matches `range` is a range, with the same rules as in a recipe.
2. Totals follow the [aggregation](#aggregation) rules, so a shopping list can show `garlic 5–7 cloves`.
3. Serialisers write ranges as `start-end` with the ASCII hyphen and no spaces, e.g. `{1-2%tbsp}`, so the file round-trips.

The check file is unchanged.

### Pantry matching

When comparing a range requirement against stock ("what can I cook?"), applications should compare against the **upper** end, so a recipe is never reported as fully covered when the cook may run short. Applications may additionally show a recipe as partially covered when stock falls between the two ends.

### Backward compatibility

Under the current specification `@garlic{2-3%cloves}` has the text quantity `2-3`. After this change it is the numeric range 2–3. Recipes that use this form already intend a range, so they gain scaling and aggregation and lose nothing. The only text whose meaning changes unintentionally is a quantity that happens to be two numbers joined by a hyphen but is not a range, such as a date or a product code in the quantity slot; these are not known to exist in practice. Quantities such as `1-2 handfuls`, `few`, `to taste` and negative numbers remain text. No migration is needed.

## Effect on applications which use Cooklang

### Specification updates

Once accepted, ranges are documented in the language specification (README and EBNF) with the canonical tests above, the `tests/README.md` description of `quantity` is extended with the `start`/`end` mapping, and the scaling, shopping list and pantry sections of the conventions gain the rules from this proposal.

### Display

1. **Ingredient lists and inline in steps** — show the range as written, with units once: `2–3 cloves`, not `2 cloves–3 cloves`. Fractions are rendered as they are for single numbers, so `1/2-1%tsp` reads `½–1 tsp`.
2. **Scaled recipes** — show the scaled range. Applications that round scaled quantities to friendly fractions round each end.
3. **Timers** — show the range, and default any countdown to the lower end.

### CookCLI (terminal and web-server)

CookCLI should:

1. Parse ranges unconditionally, without enabling any other extension. This fixes [cookcli#597](https://github.com/cooklang/cookcli/issues/597) and removes the need for `doctor` to enable the extension privately.
2. Scale both ends in `recipe read` with a scaling factor, in all output formats including `-f cooklang`, which already round-trips ranges.
3. Aggregate ranges in `shopping-list` and in the web UI's ingredient list, and read and write ranges on free-hand items and selection lines in `.shopping-list` files.
4. Expose ranges in JSON and YAML output as a value with `start` and `end`, distinct from text.
5. Compare against the upper end in pantry-based recipe matching.

No new sub-command is required.

### Mobile applications

Mobile apps should:

1. Parse ranges as numbers. The iOS app parses through the Swift bindings' `parse_recipe`, which uses the canonical (no extensions) parser preset; once cooklang-rs parses ranges with extensions disabled, no bindings change is needed. Until then, [cooklang-rs#78](https://github.com/cooklang/cooklang-rs/pull/78), which turns the extension on in the bindings, is the stopgap.
2. Scale both ends when the user changes servings. The current scaling calculator uses only the start of a range as the base quantity.
3. In cooking mode, display the timer range and start the countdown at the lower end, offering the upper end as an alternative.
4. Aggregate ranges in the shopping list following the [aggregation](#aggregation) rules, and show them as ranges. The bindings currently group quantities by value kind as well as by unit, so a number and a range of the same ingredient and unit never meet; they must be merged, with the number added to both ends.

### cook.md

The web app should parse recipes with ranges enabled so that rendered recipes, scaling and shopping lists agree with the import pipeline, which already writes ranges and already parses them as numbers in its quality checks.

### Parsers

Parsers that implement this as an extension today (cooklang-rs) should make ranges part of the core syntax, available with all extensions disabled, and update their canonical test runner for the `start`/`end` mapping. Other parsers (cooklang-ts, cooklang-swift, cooklang-c, and others) need the grammar change and the new value kind.

## Alternatives considered

### Keep ranges as text and define only how to display them

This is the status quo in half the ecosystem. It needs no parser change, but it leaves ranges unscaled and unaggregated, which is exactly the bug users report, and it does not stop the other half of the ecosystem from treating them as numbers. The inconsistency, not the display, is the problem.

### Standardise a wider "quantity expression" syntax at once

[Discussion #84](https://github.com/cooklang/spec/discussions/84) asks for alternative quantities (`{1%kg|1.25%cup}`, or `{15%oz, 1 medium can}`), and later comments suggest `~` for "about". These are worthwhile but independent: they need their own syntax decisions, none of them is implemented in a shipping parser, and bundling them would delay the one feature that is already in use and causing inconsistencies. Nothing in this proposal prevents adding them later; a range could become one of the alternatives.

### Ranges with a different unit on each end

Expressions that mix a weight and a count, such as the `@chicken wings{700%g, 6-8}` suggested in the same discussion, come up as well. They are better served by alternative quantities than by ranges, and a range with one unit is what every existing recipe and parser uses.

### Accept typographic dashes

Allowing `–` (U+2013) and `—` (U+2014) as separators would be friendlier to text pasted from websites. It was rejected to keep the grammar ASCII, like the rest of Cooklang, and because no parser accepts them today; the AI importers and editors can normalise dashes before the text reaches a file.

### Require start ≤ end

Rejecting `{3-2}` would catch a typo, but turning it into a parse error would make a whole recipe fail to load over one quantity. Keeping the ends as written and letting tools warn matches how the rest of the language treats questionable input.

### Default timers to the upper end, or to the midpoint

Starting a 25–30 minute timer at 30 minutes risks overcooking if the cook relies on the alarm; the midpoint is a number that appears nowhere in the recipe. The lower end matches how the range is used in a kitchen: check early, continue if needed.

## Acknowledgments

Thanks to [Zheoni](https://github.com/Zheoni) for designing and implementing the range values extension in cooklang-chef and cooklang-rs, where this syntax originated; to [ziofat](https://github.com/ziofat), [shicks](https://github.com/shicks), [adsilcott](https://github.com/adsilcott), [jphastings](https://github.com/jphastings) and [Earnestly](https://github.com/Earnestly) for the discussions in [#84](https://github.com/cooklang/spec/discussions/84) and [#96](https://github.com/cooklang/spec/discussions/96); to [shayaknyc](https://github.com/shayaknyc) and [adriankumpf](https://github.com/adriankumpf) for reporting [cookcli#49](https://github.com/cooklang/cookcli/issues/49) and [cookcli#597](https://github.com/cooklang/cookcli/issues/597); and to [kei-sidorov](https://github.com/kei-sidorov) for [cooklang-rs#78](https://github.com/cooklang/cooklang-rs/pull/78), which surfaced the gap in the mobile apps.
