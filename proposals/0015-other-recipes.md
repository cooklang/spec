# Reference other recipes

* Proposal: [0015-other-recipes](0015-other-recipes.md)
* Authors: [Alexey Dubovskoy](https://github.com/dubadub)
* Status: **Awaiting review**

## Introduction

This proposal introduces the ability to reference ingredients that are essentially
other recipes using relative file paths. This feature allows recipes to be modular
and reusable, enabling cooks to maintain base recipes (like sauces, dressings, or
common components) that can be referenced from multiple other recipes.

Discussion threads:
- [Include other recipes](https://github.com/cooklang/spec/discussions/55)
- [Link/Include one recipe in another?](https://github.com/cooklang/spec/discussions/117)

## Motivation

Currently, Cooklang doesn't have a built-in way to reference other recipes. This
leads to several issues:

1. Recipe duplication - Common components like sauces or marinades need to be
   copied into each recipe that uses them
2. Maintenance challenges - When a base recipe is updated, all copies need to be
   manually updated
3. Limited organization capabilities - Cannot create meta-recipes or meal plans
   (like "Thanksgiving Dinner" or "Weekly Meal Prep") that reference multiple
   individual recipes with their respective quantities

## Proposed solution

Add support for referencing other recipes using the existing @ ingredient syntax,
inferring relative file paths from the recipe name:

```cooklang
Add @./Pesto Sauce{100%g} and stir.
Add @../sauces/Bechamel{200%ml} to the pan.
Add @./Hollandaise{150%g} and mix well.
```

When the parser detects a relative path, it marks the ingredient as a reference
that can be parsed separately.

## Detailed design

Two main approaches have been considered for recipe references:

### Approach 1: Special Recipe Reference Syntax

This approach is implemented as a parser extension:

```cooklang
Add @@Boiled Icing{6%tsp} to the cake.
```

**Advantages:**
- Works in non-filesystem environments (e.g., databases)
- More sharing-friendly as it doesn't assume specific directory structures
- Simpler for non-technical users who don't understand file paths
- Allows for flexible organization as recipes can be reorganized without updating
  references

**Disadvantages:**
- Requires recipe lookup/resolution rules to be defined
- May be ambiguous when multiple recipes have the same name
- Introduces new syntax that may not be immediately intuitive
- Requires clear documentation about lookup behavior

### Approach 2: File-based References (Current Proposal)

```cooklang
Add @./Boiled Icing{6%tsp} to the cake.
Add @../sauces/Bechamel{200%ml} to the pan.
```

**Advantages:**
- Explicit and unambiguous about recipe location
- Follows familiar file system patterns
- No inference required about recipe location
- Consistent with how other systems handle file references

**Disadvantages:**
- References break when files are moved
- Less suitable for database-backed implementations
- Requires understanding of relative file paths
- May make recipe sharing more complex due to assumed directory structures

After careful consideration, this proposal recommends the file-based approach
(Approach 2) because:
1. It maintains clarity and explicitness about recipe locations
2. It follows the Unix philosophy of being explicit rather than magical
3. It allows for clear error messages when recipes can't be found
4. It enables static analysis of recipe dependencies

The parser will:
1. Identify recipe references starting with `@./`, `@../` or Windows `@.\`, `@..\`
2. Mark these ingredients as references and store their relative paths
3. Set the ingredient name as the file stem of the reference

The parser will not:
1. Validate that the relative path exists
2. Load and parse the referenced recipe
3. Apply the specified quantity to the referenced recipe

These responsibilities are left to the parser caller.

## Effect on Applications Using Cooklang

Applications should display a link to referenced recipes in ingredient lists. For
shopping lists, applications need to recursively apply quantities and extract
ingredients from referenced recipes.

### CookCLI (Terminal and Web-server)

CookCLI needs to be updated to:
1. Output an extra note indicating that an ingredient is a reference and its path
2. Adjust shopping list generation to include ingredients from referenced recipes

### Mobile Applications

Mobile apps need to:
1. Add UI elements to indicate referenced recipes with links
2. Update shopping list generation to include referenced recipe ingredients
3. Allow users to navigate to referenced recipes
4. Provide visual feedback when referenced recipes cannot be found

### Implementation Considerations

When implementing recipe references, systems should:
1. Maintain a cache of recipe locations to avoid repeated filesystem traversal
2. Provide clear error messages for missing or circular references
3. Provide tools for finding broken references when recipes are reorganized

## Alternatives Considered

1. Separate syntax for referencing recipes (`@@recipe_name{quantity}`):
   While more consistent with existing ingredient syntax, this would be more
   verbose and harder to read.

2. Using unique IDs instead of file paths:
   This would make recipes less human-readable and harder to manage manually.

3. Using an include-style syntax (like `!include`):
   The `@` syntax was chosen as it's more consistent with Cooklang's existing
   conventions for ingredients.

## Acknowledgments

Thanks to the Cooklang community members who provided feedback on the relative path
syntax and cross-platform compatibility considerations. Special thanks to
[@Zheoni](https://github.com/Zheoni) for implementing the extension that preceded
this feature.
