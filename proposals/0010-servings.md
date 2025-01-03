# Servings

* Proposal: [0010-servings](0010-servings.md)
* Authors: [Alexey Dubovskoy](https://github.com/dubadub)
* Status: **Awaiting review**


## Introduction

This proposal introduces a standardized way to specify serving sizes in recipes and scale
ingredient quantities accordingly. It allows recipe authors to define supported
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
4. There's no consistent way to indicate the intended serving size of a recipe

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

2. Two ways to specify ingredient scaling:

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

To make it scaled to 4 servings we just linearly multiply each ingredient to ratio of desired serving to default serving.

```cooklang
-- ratio = 2 (4 desired servings / 2 default servings)
-- multiply milk quanity by ratio 2

Add @milk{1%cup} and mix until smooth.
```

b. TODO, do we really need that? Manual Quanted scaling with explicit quantities for each serving size. We can give supported servings and in each ingredient specify manual amounts.

```cooklang
---
servings: 2|4|6
---

Add @milk{1|1.5|2%cup} and mix until smooth.
```

This approach provides flexibility while maintaining simplicity. Linear scaling covers most common cases, while manual scaling handles exceptions where ingredients don't scale linearly.

TODO We might go extra mile an provide some fitting function that will interpolate/extrapolate scaling for not defined values

c. Unscaled. Sometimes we need to lock ingredient amount to one value

```cooklang
---
servings: 2
---

Add @milk{=1%cup} and mix until smooth.
```

TODO play with baking percentages

## Detailed design

### Metadata Tag Syntax

The servings metadata tag follows this format:
```
>> servings: number(|number)*
```

- Numbers must be positive integers
- At least one serving size must be specified
- Multiple serving sizes are separated by the pipe character (|)
- Serving sizes should be listed in ascending order

### Ingredient Scaling Syntax

1. Linear scaling (TODO make it so existing recipes scale linearly):
```
@ingredient{quantity%unit}
```
Nothing required. Default recipe should scale linearly.

2. Manual scaling:
```
@ingredient{quantity1|quantity2|quantity3%unit}
```
- The number of quantities must match the number of serving sizes in the metadata tag
- Each quantity can be a number or fraction
- Quantities are separated by the pipe character (|)

3. Pinned ingredients:
```
@ingredient{=quantity%unit}
```
- Ingredients quanitites locked with ^ scaling notation maintain the same quantity for all serving sizes

## Effect on applications which use Cooklang

### CookCLI (terminal and web-server)

CookCLI should be updated to:
1. Add a `--servings` flag to specify desired serving size
2. Include serving information in JSON output
3. Automatically calculate scaled quantities in recipe output
4. Add validation for serving size syntax

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

2. Recipe Editor Changes:
   - Add UI for setting supported serving sizes
   - Provide interface for specifying scaled quantities
   - Add validation for serving syntax

3. Recipe List Changes:
   - Show default serving size in recipe previews
   - Allow filtering/sorting by serving size

## Alternatives considered

1. Percentage-based scaling:
```
>> servings: 100%|200%|400%
```
Rejected because absolute numbers are more intuitive for users.

2. Range-based servings:
```
>> servings: 2-8
```
Rejected because discrete values provide better control over scaling.

3. Separate scaling factors:
```
@ingredient{quantity%unit}{x2|x4|x8}
```
Rejected for being more verbose and less readable.

## Acknowledgments

Thanks to the Cooklang community members who participated in the discussions about servings support and default serving sizes, particularly in GitHub discussions #70 and #66. Their input helped shape this proposal into a more robust and user-friendly solution.