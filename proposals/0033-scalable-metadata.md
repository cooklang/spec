# Metadata that scales

* Proposal: [0033-scalable-metadata](0033-scalable-metadata.md)
* Authors: [Alexey Dubovskoy](https://github.com/dubadub)
* Review Manager: TBD
* Status: **Awaiting review**

## Introduction

This proposal says which metadata values change when a recipe is scaled. By default none do. Two canonical keys scale, `servings` and `yield`, and so does any key the author ends with a star.

```cooklang
---
title: Bread rolls
servings: 4
yield: 600%g
makes*: 12 rolls
cost*: 3.60
oven: 220
---
```

Scaled by 2, the recipe has `servings` 8, `yield` 1200%g, `makes` 24 rolls and `cost` 7.2. `oven` stays 220.

## Motivation

**Scaling leaves the front matter behind.** When a recipe is scaled, the amounts in the steps change and the metadata does not, apart from a whole-number `servings`. A doubled recipe still says `yield: 500%ml`, and an author's own `makes: 12 rolls` or `cost: 8` stays as written. With CookCLI 0.38.1, scaling by 2:

| Front matter | After scaling | Note |
| --- | --- | --- |
| `servings: 4` | `8` | the only value that scales |
| `servings: 3`, scaled by 1.5 | `5` | 4.5 is rounded |
| `servings: 6 portions` | `6 portions` | not scaled, although the conventions say the leading number is used |
| `yield: 500%ml` | `500%ml` | not scaled |
| `makes: 12`, any other key | unchanged | |

**Scaling every number is wrong too.** `rating: 4`, `year: 2022`, `oven: 220`, `prep time: 20 minutes` and `difficulty: 3` are numbers that must stay. An application cannot tell the two groups apart from the value, so the author has to say which keys follow the recipe.

**Nutrition has been asked for.** A [CookCLI feature request](https://github.com/cooklang/cookcli/issues/299) asks for nutrition values that are multiplied with the servings, and notes that other keys could use the same thing. This proposal gives the general rule and says what it means for nutrition.

## Proposed solution

Add to the scaling section of `conventions.md`:

> Metadata does not scale, with two exceptions: the canonical keys `servings` (or `serves`) and `yield`, and any key whose name ends with a star (`*`). When a recipe is scaled, the number at the start of such a value is multiplied by the same factor as the amounts in the recipe. The star is a mark and not part of the key's name.

## Detailed design

1. **The default.** A metadata value is never changed by scaling unless one of the two exceptions applies.

2. **Built-in keys.** `servings`, its other name `serves`, and `yield` always scale. They need no star, and a star on them changes nothing.

3. **The mark.** A key scales when the last character of its name is `*`: `makes*: 12`. The parser returns the key as `makes`, with a flag that it scales. Applications show `makes`, not `makes*`.

4. **Why the end of the key.** In YAML, a `*` at the start of a plain key or value is an alias. `*makes: 12` is an error, and a parser that meets it loses the whole front matter. A star at the end needs no quotes.

5. **A star on a canonical key.** The key keeps its meaning: `prep time*: 20 minutes` is still the preparation time, and it now scales. That is the author saying the time grows with the batch.

6. **Mappings and lists.** A star on a mapping covers every value under it. A star on a key inside a mapping covers that key only. In a list, every item scales.

7. **What scales in a value.** Only the number at the start:

   | Value | Scaled by 2 |
   | --- | --- |
   | a number: `12`, `3.60` | `24`, `7.2` |
   | a number, then text: `12 rolls`, `6 portions` | `24 rolls`, `12 portions` |
   | a Cooklang amount: `500%ml` | `1000%ml` |
   | a fraction or a range: `1/2 cup`, `2-3` | `1 cup`, `4-6` |
   | a list: `[500%ml, 350%g]` | each item |
   | anything else | unchanged |

   In `12 rolls (2 trays)` the 2 stays. Two numbers that both scale are two keys.

8. **No conversion and no rounding.** `600%g` becomes `1200%g`, not `1.2%kg`, and `servings: 3` scaled by 1.5 is 4.5. An application may convert or round what it shows. Rounding in the parser loses the recipe as written: 3 becomes 5, and scaling back from 5 does not return 3.

9. **A star that does nothing.** `title*: Rolls` has no number to scale. It is not an error; a checking tool may report it.

10. **One factor.** Every way of scaling ends in one factor, and it applies to this metadata exactly as it applies to the amounts in the steps: a plain factor (`@./rolls{2}`), a servings target (`@./rolls{8%servings}`, factor = target ÷ `servings`) and a yield target (`@./stock{250%ml}`, factor = target ÷ the matching `yield` entry). An application that reads the metadata of a referenced recipe, such as a menu that adds up `cost*` for the week, gets the scaled values.

11. **No lock.** Amounts in steps use `=` to stay fixed. Metadata does not need it, because staying fixed is the default.

12. **With hidden keys.** A key can carry both marks from [proposal 0031](https://github.com/cooklang/spec/pull/171): `.cost*` is the hidden custom key `.cost`, and it scales.

### Nutrition

Nutrition values written per serving do not scale. The number for one serving is the same however many servings are cooked. The total for the batch is the value per serving multiplied by `servings`, and `servings` scales, so the total follows the recipe with no rule of its own. A recipe with no `servings` has nothing to carry the factor, so an application treats the recipe as written as one serving and multiplies by the factor.

A recipe with `servings: 6` and 539 kcal per serving, scaled to 12 servings:

| | Per serving | For the batch |
| --- | --- | --- |
| As written | 539 kcal | 3,234 kcal |
| Nutrition not scaled (proposed) | 539 kcal | 6,468 kcal |
| Nutrition scaled with the recipe | shows 1,078 kcal beside "12 servings" | a reader takes it as per serving, or divides and gets 90 |

Recipes imported from web pages already hold nutrition per serving, because that is what the pages publish. The example in the feature request (`servings: 1`, 100 kcal, 300 kcal for three servings) comes out the same under this rule: 100 per serving, 300 for the batch.

The names, units and shape of a `nutrition` key are not part of this proposal.

### Canonical tests

To be added with the conventions change. A parser returns the key without the star, and scaling changes the leading number:

```yaml
testMetadataScalableKey:
  source: |
    ---
    servings: 4
    makes*: 12 rolls
    oven: 220
    ---
  scale: 2
  result:
    steps: []
    metadata:
      "servings": 8
      "makes": 24 rolls
      "oven": 220
```

The test format has no `scale` field today; it is shown here to state the expected result.

## Effect on applications which use Cooklang

Parsers strip a star from the end of a key before looking the key up, keep a flag, and scale `yield` and flagged keys together with `servings`. `yield` has to be its own key first, which is [proposal 0017](https://github.com/cooklang/spec/pull/146).

**Files with starred keys in applications that do not know the rule yet.** Nothing fails. The key is shown with the star in its name (`makes*: 12 rolls`) and the value does not change when the recipe is scaled, which is what happens to every custom key today.

**Fractional servings.** Applications that show `servings` have to accept a number with a decimal, and decide how to show it.

### CookCLI (terminal and web-server)

`cook recipe` and the web server's recipe page show starred keys by their name without the star, with the scaled value. `cook doctor` reports a star on a value with no number, and a `*` at the start of a key.

No new sub-command or flag.

### Mobile applications

The servings control gives the factor; the recipe screen shows the scaled `yield` and starred keys beside the scaled amounts.

## Alternatives considered

**Scale every value written as an amount** (`500%ml`, `100%kcal`), with a mark to lock the ones that should stay. It needs no star, but it scales values nobody meant to scale, and it does nothing for plain numbers such as `makes: 12`. Off by default is the safer rule for files that already exist: nothing in them changes.

**A star at the start of the key** (`*makes: 12`). It reads well, but it is a YAML alias. It has to be written `"*makes": 12` every time, and a forgotten quote loses all the metadata of the file.

**Another character at the start** (`+makes: 12`). It needs no quotes. A star is the more familiar sign for multiplying.

**A mark on the value** (`makes: 12*`, `makes: *12`). The second form has the same YAML problem. Both turn a number into text for every tool that does not know the rule.

**A list of keys that scale** (`scales: [makes, cost]`). It keeps key names clean, but the list and the keys can drift apart, and the mark is away from the value it changes.

**Scaling nutrition.** See the table above: a per-serving value that grows with the servings is wrong beside the servings count. A switch inside the mapping (`per: recipe`) could mark values that are totals for the whole batch and do scale; it belongs with the proposal that defines the nutrition key.

**Rounding `servings` to a whole number**, as today. It hides a factor that the amounts in the steps still carry.

## Open questions

- **Where the mark goes.** The end of the key, as proposed, or a quoted star or a `+` at the start?
- **Nutrition.** Per serving and never scaled, as proposed, or a switch for whole-batch totals?
- **`yield`.** Should it scale without a star?
- **`servings` with text and with decimals.** Should `6 portions` scale by its leading number, and should 4.5 servings stay 4.5?
- **A star on a canonical key.** Allowed, as proposed, or only on custom keys?
- **Mappings.** Should a star on a mapping cover everything under it?

## Acknowledgments

Thanks to the author of the CookCLI feature request for raising scaling of metadata and for noting that it is wider than nutrition.
