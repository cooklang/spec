# Servings

* Proposal: [0010-servings](0010-servings.md)
* Authors: [Alexey Dubovskoy](https://github.com/dubadub)
* Status: **Accepted, in implementation**

## Introduction

This proposal introduces a standardized way to specify serving sizes in recipes and scale
ingredient quantities appropriately. It allows recipe authors to define supported
serving sizes and automatically or manually adjust ingredient quantities, making
it easier for users to adapt recipes to their needs.

Discussion thread: [Is servings supported?](https://github.com/cooklang/spec/discussions/70)
and [Default serving size](https://github.com/cooklang/spec/discussions/66).

## Motivation

Currently, Cooklang lacks a standardized way to handle recipe scaling
based on serving sizes. This leads to several issues:

1. Users need to manually calculate ingredient quantities when scaling recipes up or down
2. Recipe authors can't specify non-linear scaling relationships for ingredients
3. Applications can't automatically adjust quantities based on desired servings

These limitations make it harder for users to adapt recipes to their needs
and for applications to provide serving-related features.

## Proposed solution

The solution utilizes two main components:

1. A metadata tag for defining supported serving sizes (introduced in
https://github.com/cooklang/spec/blob/main/proposals/0007-canonical-metadata.md):

```cooklang
---
servings: 2
---
```

If metadata tag isn't present we assume that servings is 1. The ingredients in a recipe are
scaled to this specified servings size.

2. Three ways to specify ingredient scaling:

a. Default linear scaling. Regular Cooklang recipe should scale each ingredient linearly:

```cooklang
Add @milk{1/2%cup} and mix until smooth.
```

This will use defined servings and rescale appropriately. Imagine having this recipe:

```cooklang
---
servings: 2
---

Add @milk{1/2%cup} and mix until smooth.
```

To make it scaled to 4 servings we just linearly multiply each ingredient to ratio
of desired serving to default serving.

```cooklang
-- ratio = 2 (4 desired servings / 2 default servings)
-- multiply milk quanity by ratio 2

Add @milk{1%cup} and mix until smooth.
```

b. Unscaled. Sometimes we need to lock ingredient amount to one value

```cooklang
---
servings: 2
---

Add @milk{=1%cup} and mix until smooth.
```

All these options can be mixed together in one recipe.

### Baking percentages

In this setup baking percentages should just work.

```cooklang
Mix @flour{1000%g} and @water{600%g}.
```

Users should be able to calculate ratio factor knowing how much flour they want to use. Ratio = 600 / 1000 = 0.6.

### Timers

Timers don't scale. Seems that in most cases portion size stays the same
and hence cooking time should stay the same. Of course there're recipes
that need timer scaling for example turkey roasting, but in that case we can
leave it up to user to set the proper timer.

### Cookware

Cookware doesn't scale.

## Detailed design

### Metadata Tag Syntax

The servings metadata tag follows this format:
```
>> servings: number
```

### Ingredient Scaling Syntax

1. Linear scaling
```
@ingredient{quantity%unit}
```
Nothing required. Default recipe should scale linearly.

2. Pinned ingredients:
```
@ingredient{=quantity%unit}
```
- Ingredients quanitites locked with `=` scaling notation maintain
the same quantity for all serving sizes.

## Effect on applications which use Cooklang

### CookCLI (terminal and web-server)

CookCLI should be updated to:
1. Add a `--servings` flag to specify desired serving size on recipe
   read and shopping list generation
2. Include serving information in JSON output
3. Automatically calculate scaled quantities in recipe output

Example CLI usage:
```bash
cook recipe read recipe.cook --servings 4
```

### Mobile applications

Mobile apps should add:

1. Recipe View Changes:
   - Add serving size selector
   - Display current serving size
   - Show scaled ingredient quantities

2. Recipe List Changes:
   - Show default serving size in recipe previews

3. Shopping List Changes:
   - Show serving size in shopping list
   - Show scaled quantities in shopping list

## Alternatives considered

Originally thinking about having manual scaling with explicit quantities
for each serving size. We can give supported servings and in each ingredient specify manual amounts.

```cooklang
---
servings: 2|4|6
---

Add @milk{1|1.5|2%cup} and mix until smooth.
```

Linear scaling covers most common cases, while manual scaling handles exceptions where
ingredients don't scale linearly.

I don't think there would be usage for this, but amount of complexity it adds is too high.
Also we would need to show users if metadata doesn't match the number of quantities in the recipe.

In the future we might go extra mile an provide way to specify scaling function that
will calculate scaling when non-linear scaling is needed.

## Acknowledgments

Thanks to the Cooklang community members who participated in the discussions about servings
support and default serving sizes, particularly in GitHub discussions [Is servings supported?](https://github.com/cooklang/spec/discussions/70)
and [Default serving size](https://github.com/cooklang/spec/discussions/66). Their input
helped shape this proposal into a more robust and user-friendly solution. Also huge thanks to
[@Zheoni](https://github.com/Zheoni) for implementing this feature.
