# Baker's Percentages and Named Ingredient Bases

- Proposal: [0018-Bakers-Percentages](0018-bakers-percentages.md)
- Authors: [Teddy Cleveland](https://github.com/spiicy-sauce)
- Review Manager: TBD
- Status: **Awaiting review**

## Introduction

I propose adding support for **named ingredient bases** (basises?), allowing authors to
declare a pool of ingredients (e.g. `flour`) with a fixed total, and express
ingredient quantities as percentages of that pool; this approach is used
heavily in baking and fermentation and is commonly known as **baker's
percentages**.

Discussion thread: [TBD](https://github.com/cooklang/spec/discussions).

## Motivation

I've spent the past few years on-and-off trying to build applications to
represent baking recipes ([breadlikewatercolor.com](https://breadlikewatercolor.com/about), and an [interactive recipe builder](https://blwc-xi.vercel.app/)), but I've always wished they fit into a broader recipe
ecosystem. I wanted to see if Cooklang could be extended to support this use
case, as it doesn't have a way to express this approach to recipe building as
far as I could tell, but with a small addition it could!

Baker's percentages are the standard notation for bread formulas. `75% water` water means the water should be 75% of the weight of all the flour in the recipe, regardless of batch size: the ratio _is_ the formula in this sense. Scaling to a different batch size means changing one number: the basis total, and all ingredient quantities can be automatically computed.

I came across some related prior discussions in preparing this proposal:
[#53](https://github.com/cooklang/spec/discussions/53),
[#79](https://github.com/cooklang/spec/discussions/79).

## Proposed solution

**New front-matter key `basis`** — declares named bases, each with
a fixed `total` and an optional `members` list:

```yaml
basis:
  flour:
    total: 500%g
    members: [bread flour, whole wheat flour]
```

The declared `total` is the authoritative denominator for all `%basis-name`
expressions in the recipe. `members` is a lookup pattern matched against
ingredient names in the recipe steps — a name in `members` that doesn't appear
in the recipe is a parser warning, not a silent failure. This centralized
approach also avoids any issues of mismatched units within the recipe.

Ingredients are resolved by name lookup against `members`:

- **Constituent** (name in `members`): part of the basis; its resolved quantity
  counts toward the total.
- **Reference** (name not in `members`): expressed relative to the basis but
  not part of it (e.g. water as a percentage of flour).

Both resolve identically: `quantity = percentage × total`. The distinction
enables parsers to validate that constituents sum to `total`, and also to validate
that constituent percentages sum to 100%.

```cook
---
basis:
  flour:
    total: 500%g
    members: [bread flour, whole wheat flour]
yield: 1%loaf
---
Mix @whole wheat flour{20%flour} and @bread flour{80%flour}.
-- Constituents (in members): resolve to 100g and 400g.
-- Change total to 1000%g and both scale instantly.
Add @water{75%flour} and @salt{2%flour} and @starter{20%flour}.
-- References (not in members): resolve to 375g, 10g, and 100g.
```

side note: `basis.total` is intentionally distinct from `yield`. `total` is the input budget, while `yield` is the output unit of the finished product used for cross-recipe scaling.

## Detailed design

**Grammar:** No change required. The existing
`amount = quantity | (quantity, "%", units)` rule already covers basis-relative
amounts; the semantic interpretation of `units` is extended when it matches a
declared basis name.

**Basis declaration:** Each basis MUST declare a `total` (a `quantity%unit`
amount) and optionally a `members` list of ingredient name strings. Recipes without a `basis`
key are unaffected.

**Validation:** Parsers MUST warn if a name in `members` does not match any
ingredient used in the recipe steps. Parsers SHOULD warn if constituent
percentages don't sum to 100%.

**Resolution:** When a `%basis-name` amount is encountered, the parser resolves
`quantity = percentage × total`, then checks whether the ingredient name appears
in `members` to classify it as a constituent or reference. No ordering
constraint is introduced — `total` is always known before any ingredient is
resolved.

**Example:**

```cook
---
title: Basic Sourdough
yield: 900%g
basis:
  flour:
    total: 500%g
    members: [bread flour, whole wheat flour]
---
Mix @bread flour{80%flour} and @whole wheat flour{20%flour}.
Add @water{75%flour} and @salt{2%flour} and @starter{20%flour}.
Autolyse ~{30%minutes}, bulk ferment ~{4%hours}, cold proof ~{12%hours}.
Bake in #dutch oven{} ~{20%minutes} covered, ~{20%minutes} uncovered.
```

Resolved ingredient list:

| Ingredient  | Quantity | % flour |
| ----------- | -------- | ------- |
| bread flour | 400 g    | 80%     |
| whole wheat | 100 g    | 20%     |
| water       | 375 g    | 75%     |
| salt        | 10 g     | 2%      |
| starter     | 100 g    | 20%     |

## Effect on applications which use Cooklang

### CookCLI (terminal and web-server)

JSON output should gain a `bases` key with each basis's `total`, its
constituent ingredients (matched via `members`), and a `basis_percent` field
per ingredient. Human-readable output should display a `% basis` column
alongside absolute quantities when a `basis` is declared. No new sub-command
needed.

### Mobile applications

Ingredient lists should show `% basis` alongside absolute quantities, with both
updating automatically when `basis.total` is adjusted.

## Alternatives considered

**Comments only.** Authors can write `@water{375%g} -- 75%` today. Not
machine-readable; tools cannot derive or scale the relationship.

**Inferred basis total.** Without a declared `total`, there is no denominator
to resolve `%basis` amounts against until all constituent quantities have been
parsed. Requiring `total` enables single-pass resolution.
