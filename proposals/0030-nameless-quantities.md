# Nameless quantities

* Proposal: [0030-nameless-quantities](0030-nameless-quantities.md)
* Authors: [Alexey Dubovskoy](https://github.com/dubadub)
* Review Manager: TBD
* Status: **Awaiting review**

## Introduction

This proposal adds a way to write an amount in a step that belongs to no ingredient: `@{8}`. The amount scales with the recipe and converts between unit systems as an ingredient amount does, and it is on no list.

```cooklang
Divide the @dough{} into @{8} pieces of about @{=60%g} each.
```

When the recipe is doubled this reads "into 16 pieces of about 60 g each".

Discussion: [#96](https://github.com/cooklang/spec/discussions/96).

## Motivation

**Numbers in the step text do not scale.** A recipe scales its ingredient amounts and leaves every other number as it was written. "Divide the dough into 4 pieces" still says 4 when the recipe is doubled. The cook does the sum in their head, or ends up with pieces twice the size and the wrong baking time.

**Amounts in the step text do not convert.** "Separate the dough into 20–25 g pieces" stays in grams for a reader whose application shows ounces, because the amount is plain text.

Both were asked for in discussion #96: conversion of `20g-25g` pieces when it was opened, and scaling of "split the dough into 2 sections" later. One alternative parser already has this as an extension.

It is not rare. In a set of about 16,000 converted recipes, about 5% have at least one count for the whole batch after a cutting or shaping verb ("divide the dough into 9 pieces", "cut into 12 bars").

**There is no way to write it today.** With CookCLI 0.38.1:

| Step text | Result |
| --- | --- |
| `Divide into @{4} pieces.` | the recipe is rejected: "Invalid ingredient name: is empty" |
| `Cut into {4} pieces.` | plain text, braces shown |
| `Wait ~{5%minutes}.` | a timer with no name: "Wait 5 minutes." |

A timer may have no name. An ingredient may not.

## Proposed solution

`@` followed at once by braces that hold an amount is a nameless quantity.

```cooklang
Divide the @dough{} into @{8} pieces of about @{=60%g} each.

Freeze in @{4} portions of @{=500%ml}.

Cut into @{12} slices.
```

- It scales by the same factor as the ingredients, unless it starts with `=`.
- An amount with a unit is converted as an ingredient amount is.
- It is shown in the sentence, where it is written.
- It is not an ingredient: it is on no ingredient list and no shopping list.

## Detailed design

1. **Form.** `@{` with nothing between the `@` and the brace. A space ends it: `@ {4}` is text. This is the form of the timer with no name, `~{…}`.

2. **Inside the braces.** Whatever the braces of an ingredient can hold: a number, a fraction, a range, a unit after `%`, and `=` in front to stop scaling. A proposal that changes how an ingredient amount is written changes this in the same way; there is no separate rule.

3. **Scaling.** The amount is multiplied by the same factor as the ingredients. With `=` it is not. There is no rule by unit: `@{2%cm}` scales and `@{=2%cm}` does not.

4. **No rounding.** `@{3}` at half is 1.5, as `@eggs{3}` is. An application may round what it shows. The parser does not.

5. **Conversion.** An amount with a unit is converted to the reader's unit system by the same rules and the same setting as an ingredient amount.

6. **Lists.** A nameless quantity is on no list: not the ingredient list, not the shopping list, not a pantry check, not nutrition.

7. **Display.** The amount and its unit are shown in the sentence in place of the markup. An application that prints ingredient amounts apart from the sentence still prints this one in the sentence, because it has no name to stand for it.

8. **Where.** In steps. Notes are plain text and stay so in this proposal.

9. **What is not valid.**

   | Written | Result |
   | --- | --- |
   | `@{}` | error, as today: there is neither a name nor an amount |
   | `@{some}` | warning; the text inside is shown. There is nothing to scale or convert |
   | `@?{4}`, `@-{4}`, `@&{4}` | error: a modifier needs a name |
   | `@{4}(halved)` | the amount is read; the bracket is text. There is no preparation note |
   | `#{2}` | error, as today. Cookware keeps its name |

10. **`servings` and `yield`.** There is no link. `yield: 12%slices` and "cut into `@{12}` slices" are two places that agree. Both scale by the same factor, so they go on agreeing.

### Which amounts to lock

The two requests in the discussion want opposite things, and `=` is what tells them apart.

| The amount is | Written | When the recipe is doubled |
| --- | --- | --- |
| a count for the whole batch | `into @{8} pieces` | 16 |
| the size of one piece | `of @{=60%g} each`, `@{=2%cm} thick` | unchanged; converted only |
| a total that grows with the batch | `about @{1.2%kg} of dough` | 2.4 kg |
| a temperature | `@{=180%°C}` | unchanged; converted only |

A count for each item ("cut each into 8 wedges") needs no markup. It has no unit to convert and must not scale.

A temperature with no `=` would be scaled, which is never meant. A linter should report it.

### Grammar

Added to the items a step may hold:

```ebnf
nameless quantity = "@", "{", amount, "}" ;
```

Unlike the timer with no name, the amount is required. The exact text follows acceptance.

### What a parser returns

A new item type in a step, with a quantity and units. It is not in the list of ingredients.

```yaml
source: |
  Divide the @dough{} into @{8} pieces of @{=60%g} each.
result:
  steps:
    - - type: text
        value: "Divide the "
      - type: ingredient
        name: "dough"
        quantity: "some"
        units: ""
      - type: text
        value: " into "
      - type: quantity
        quantity: 8
        units: ""
      - type: text
        value: " pieces of "
      - type: quantity
        quantity: 60
        units: "g"
        scalable: false
      - type: text
        value: " each."
```

The name of the item type and of the `scalable` field are a suggestion. A parser that already has a place for amounts found in the step text can use it.

### Backward compatibility

Every valid recipe stays valid and reads the same: `@{` with an amount is an error today, so no existing recipe can contain it.

The other direction is the hard part. A parser released before this change rejects the whole recipe when it meets `@{4}`. The recipe does not open; the reader does not see the markup as text. So:

- parsers and applications should read the form before any tool writes it;
- converters and other tools that write recipes should not write it until the applications that read it are the ones most people run;
- an author who writes it by hand in a shared recipe breaks it for readers on old applications. The specification should say so while the change is new.

A parser can read the form and show the amount in one release, and add scaling and conversion later.

### Relation to other proposals

- **0009, temperature syntax.** A temperature found in the sentence with no markup is converted and never scaled. A nameless quantity is the marked form and can scale. `@{=180%°C}` works in a parser that does no detection.
- **0021, alternative units,** and **0024, optional `%`.** They change what an amount can hold, so they apply here too: `@{=2%oz|60%g}`, `@{=60 g}`.
- **0025, alternatives.** An option with no name is written `|{300%g}`, with no `@` and only after a `|`. It is not a nameless quantity. `@{4}|@{6}` is not a group of alternatives.
- **0020, hidden components.** `@-water{2%l}` is an ingredient that is scaled and kept off the lists. It has a name and counts as an ingredient. A nameless quantity is not an ingredient at all.
- **0019, component variables.** Its amounts are divided between named ingredients. A nameless quantity has no name and takes no part.

### Canonical cases

At a factor of 2 unless said otherwise.

| Source | Result |
| --- | --- |
| `into @{4} pieces` | "into 8 pieces"; no ingredient |
| `@{4}` at 0.5 | 2 |
| `@{3}` at 0.5 | 1.5 |
| `@{20-25%g}` | 40–50 g |
| `@{=20-25%g}` | 20–25 g |
| `@{1/2%tsp}` | 1 tsp |
| `@ {4}` | text "@ {4}" |
| `@{}` | error |
| `@{some}` | warning; text "some" |
| `@?{4}` | error |
| `@{4}(halved)` | 8, then text "(halved)" |
| `@dough{} into @{8}` | one ingredient, dough; one nameless quantity |
| `#{2}` | error |
| `~{5%minutes}` | unchanged: a timer |

## Effect on applications which use Cooklang

### Specification updates

A section in the README, the EBNF rule, and the canonical tests above. No existing canonical test changes its result.

### Display

The amount is shown in the sentence, styled as an amount. It changes with the scale control and with the unit setting.

### Lists and scaling

Nothing is added to any list. Scaling has one more set of amounts to multiply.

### Diagnostics

An error for `@{}` and for a modifier with no name. A warning for a text value. A linter warning for a temperature unit with no `=`.

### CookCLI (terminal and web-server)

The terminal output prints ingredient amounts on a line under the step. A nameless quantity is printed in the sentence.

### Mobile applications

The step and cook mode show the amount in the sentence.

### Editors, the language server and syntax highlighting

`@{…}` is highlighted as an amount and is no longer reported as an error.

### Converters and other writers

Nothing at first; see "Backward compatibility". Later, a converter can write `@{N}` for a batch count it is sure of.

### Other parsers

A parser that does not know the form should, at the least, show the text inside the braces and go on.

## Open questions

1. **What scales.** Proposed: everything scales unless it has `=`, as ingredients do. The other option: an amount with a unit does not scale and a bare number does. It fits most cases ("60 g each", "2 cm") and saves most of the `=`. It is also a second rule to learn, and a total such as "about `@{1.2%kg}` of dough" would need a new mark to make it scale.
2. **Rounding a count.** Proposed: none. The other option is to round an amount with no unit to a whole number of at least 1, which hides the problem at a factor of 0.5 and gives pieces of another size without saying so.
3. **A temperature with no `=`.** Proposed: it scales, and a linter reports it. The other option is that a temperature unit never scales, which puts a unit table into the scaling rule.
4. **Text values.** Proposed: `@{some}` is a warning and is shown as text. It could be an error.
5. **Notes.** Proposed: steps only. "Makes `@{12}` discs" in a note is a fair wish, and it needs notes to be parsed first.
6. **Cookware.** Proposed: `#{2}` stays an error. Nobody has asked for it.
7. **When tools may write it.** How long after parsers read the form may converters start to write it?

## Alternatives considered

- **Bare braces, `{4}`.** The shortest form, and an old parser shows it as text and does not fail. Braces with no sigil are ordinary text today, and proposal 0025 uses `|{300%g}` for an option with no name. A sigil keeps "this is markup" visible.
- **`%{4}`.** Suggested in the discussion. `%` already stands between a number and its unit.
- **`{{4}}`, with an optional name, `{{name:4%unit}}`.** Used by the alternative parser mentioned above. It is a new bracket form to learn where `@{…}` reuses one that exists, and the name opens the question of what it refers to.
- **A new sigil.** `^{…}` and `!{…}` were suggested for temperatures. One more sigil for something that is, in form, an ingredient amount with the name left out.
- **Detection with no markup.** Reading "into 4 pieces" from the sentence. A parser cannot tell a batch count from a count for each item, or from "step 4".
- **A name for use elsewhere and arithmetic between quantities** ("each piece is the dough weight divided by 8"). Left out; this proposal covers the written number only.

## Acknowledgments

Earnestly opened discussion #96. jphastings and FranGarc discussed the form and whether it is worth a mark. yutotakano gave the scaling case, and tmlmt built it first in an alternative parser.
