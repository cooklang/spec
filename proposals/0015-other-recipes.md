# Reference other recipes

* Proposal: [0015-other-recipes](0015-other-recipes.md)
* Authors: [Alexey Dubovskoy](https://github.com/dubadub)
* Status: **Awaiting review**

## Introduction

This proposal introduces the ability to reference ingredients
from other recipes using relative file paths. This allows recipes
to be modular and reusable, enabling cooks to maintain base recipes
(like sauces, dressings, or common components) that can be referenced
from multiple other recipes.

Discussion thread: [Include other recipes](https://github.com/cooklang/spec/discussions/55) and
[Link/Include one recipe in another?](https://github.com/cooklang/spec/discussions/117)

## Motivation

Currently, Cooklang doesn't have a built-in way to reference other recipes.
This leads to several issues:

1. Recipe duplication - Common components like sauces
   or marinades need to be copied into each recipe that uses them
2. Maintenance challenges - When a base recipe is updated,
   all copies need to be manually updated
3. Lack of modularity - Cannot break down complex
   recipes into reusable components

## Proposed solution

Add support for referencing other recipes using relative file paths with the `@` syntax:

```cooklang
Add @./Pesto Sauce{100%g} and stir.
Add @../sauces/Bechamel{200%ml} to the pan.
Add @./Hollandaise{150%g} and mix well.
```

## Detailed design

The parser will:
1. Identify recipe references starting with `@`
2. Validate the relative path exists
3. Load and parse the referenced recipe
4. Apply the specified quantity to the referenced recipe
5. Include the referenced recipe's ingredients in shopping lists with adjusted quantities

The grammar should be extended to support the following syntax:

```ebnf:proposals/0015-other-recipes.md
recipe_reference ::= '@' relative_path '{' quantity '}'
relative_path ::= ('./' | '../' | '.\' | '..\') path_component
path_component ::= [a-zA-Z0-9_-]+ ('/' | '\' path_component)*
```

## Effect on applications which use Cooklang

Changes in Cooklang syntax often require supplementary UI changes in
apps which use it (otherwise why these changes if nothing will
use them). Describe in details what changes should be implemented.
The detail in this section should be sufficient for someone
who is *not* one of the authors to be able to reasonably implement
the feature.

### CookCLI (terminal and web-server)

CookCLI needs to be updated to:
1. Support resolving relative paths when parsing recipes
2. Adjust the shopping list generation to include ingredients from referenced recipes
3. Add a new flag `--expand-references` to optionally show full contents of referenced recipes

### Mobile applications

Mobile apps need to:
1. Add UI elements to indicate referenced recipes (e.g., links or expandable sections)
2. Update shopping list generation to include referenced recipe ingredients
3. Allow users to navigate to referenced recipes
4. Provide visual feedback when referenced recipes cannot be found

## Alternatives considered

1. **Absolute paths only**: Considered only allowing absolute paths from recipe root, but this would make recipe collections less portable.

2. **Recipe IDs**: Considered using unique IDs instead of file paths, but this would make recipes less human-readable and harder to manage manually.

3. **Include syntax**: Considered using an include-style syntax like `!include`, but the `@` syntax is more consistent with Cooklang's existing conventions for ingredients.

## Acknowledgments

Thanks to the Cooklang community members who provided feedback on the relative path syntax and cross-platform compatibility considerations.
