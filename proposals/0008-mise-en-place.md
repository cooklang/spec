# Mise en place

* Proposal: [0008-mise-en-place](0008-mise-en-place.md)
* Authors: [Alexey Dubovskoy](https://github.com/dubadub)
* Status: **Awaiting review**


## Introduction

In professional kitchens, ["mise en place"](https://en.wikipedia.org/wiki/Mise_en_place) (everything in its place) is a core principle. Similarly, in Cooklang, we can optimise the workflow by introducing a shorthand syntax for ingredients, allowing recipes to be written more concisely. By enabling this syntax, users can avoid repetitive and redundant instructions, improving readability and ease of use.

Discussion thread: [Allow for short hand preparations in ingredients list to not double count](https://github.com/cooklang/spec/discussions/57) and [Ingredient variants shorthand](https://github.com/cooklang/spec/discussions/74).

Extensions link: [Extensions](https://github.com/cooklang/cooklang-rs/blob/main/extensions.md#component-note)

## Motivation

One of Cooklang's main goals is to create a recipe format that is both human-readable and easy to write. However, many recipes require the repetition of common ingredient preparations (such as peeling, chopping, or marinating). This redundancy can result in unnecessarily lengthy instructions. For instance, the following two lines:

```cooklang
Peel and finely chop @onion{1}. Peel and mince @garlic{2%cloves}.

Mix onion and garlic into paste.
```

While clear, the repetition of ingredient preparations creates clutter. A more efficient approach would be to allow shorthand notation for ingredient preparations, enabling us to condense the recipe while maintaining clarity.

## Proposed solution

The solution involves allowing users to define common ingredient preparations within the ingredient reference itself, using a shorthand syntax. This means that instead of writing out detailed preparation instructions multiple times, users can append these instructions directly to the ingredient references. This reduces redundancy without losing any important information.

For example:

**Current syntax:**

```cooklang
Peel and finely chop @onion{1}. Peel and mince @garlic{2%cloves}.

Mix onion and garlic into paste.
```

**Shorthand syntax (proposed):**

```cooklang
Mix @onion{1}(peeled and finely chopped) and @garlic{2%cloves}(peeled and minced) into paste.
```

In this approach, the preparations are attached to the ingredient references, reducing the need to repeat them.

## Detailed design

The Cooklang grammar would need to support this new shorthand syntax by parsing the parentheses following an ingredient reference as part of the ingredient specification.

We shouldn't parse shorthand preparations if there's a whitespace between ingredient reference and parentheses.

## Effect on applications which use Cooklang

### CookCLI (terminal and web-server)

CookCLI recipe output should contain ingredients with preparations.

```
Ingredients:
  garlic, peeled and minced          3 cloves
  onion, peeled and finely chopped   1
  ...
```

Similar change is required for the web-server.

### Mobile applications

Mobile apps will need UI adjustments to extra ingredient information in the recipe details. Also it would be beneficial to add an extra cooking step before all the steps where we request the user to gather and prepare required ingredients.


## Alternatives considered

1. Add it inside quantity block after units using comma:

```
When the sauce has thickened, stir in the @fresh basil{1/4%cup, chopped}.
```

or pipe:

```
Combine @oats{6%cups|steel-cut}, @raisins{1%cup|golden}, and @coconut{2%cups|shredded} in a #bowl.
```

This approach may conflict with a future servings proposal where we need to use multiple quantities.

## Acknowledgments

Special thanks to the community members who have contributed to discussions about shorthand ingredient preparation. The feature has been implemented by [@Zheoni](https://github.com/Zheoni) as parser extension.
