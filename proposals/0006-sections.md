# Sections

* Proposal: [0006-sections](0006-sections.md)
* Authors: [Alexey Dubovskoy](https://github.com/dubadub)
* Status: **Accepted, in implementation**

## Introduction

This proposal aims to introduce a new block type to the Cooklang language called "section." Sections a common structure blocks within recipes, allowing users to group and organize content more intuitively. By breaking recipes into named, clearly defined segments, the Cooklang language can improve readability, usability, and content structure, especially for more complex recipes.

**Discussion threads**: [Break down recipe instructions further](https://github.com/cooklang/spec/discussions/59), [Functionality for splitting into components and listing component ingredients](https://github.com/cooklang/spec/discussions/72), and [Description, notes, and metadata...](https://github.com/cooklang/spec/discussions/46).

**Extensions link**: [Extensions](https://github.com/cooklang/cooklang-rs/blob/main/extensions.md#section-block)


## Motivation

In its current form, Cooklang organises recipe instructions and ingredients linearly, limiting the ability to express hierarchical or segmented recipe components. For complex recipes that involve separate steps (e.g., "Dough," "Filling," "Topping") or variations within a recipe (such as optional garnish sections), a flat format can make the instructions harder to follow.

In these cases, while part of a single recipe, sections could be prepared separately—for instance, making dough the night before to allow it to rest. This means that when working on the dough, the user only needs to gather the ingredients related to that section.

Currently, users can attempt to work around this by manually adding section titles to the instructions, but this approach introduces extra steps and does not allow for selecting section-specific ingredients easily.

By introducing a formal syntax for sections, we can:

1. Improve the readability and maintainability of complex recipes.
2. Enable the preparation of specific parts of a recipe independently.
3. Lay the foundation for future features, such as cross-referencing sections.

In summary, adding sections provides a structured approach to organising complex recipes, making them easier to write, read, and implement across Cooklang-compatible applications.

## Proposed solution

We propose introducing a new syntax to Cooklang that defines sections as explicit blocks. The syntax would look like this:

```cooklang
= Dough

Mix @flour{200%g} and @water{50%ml} together until smooth.

== Filling ==

Combine @cheese{100%g} and @spinach{50%g}, then season to taste.
```

In this example, each `= Section` header represents a distinct logical group within the recipe. The number of `=` characters does not matter, and the closing `=` is optional. The text of the section title is not parsed for ingredients or cookware.

A section ends either when the next section begins or at the end of the file.


## Detailed design

The new syntax for sections introduces `=` as a section identifier repeated at least once, followed by the optional section name. Optionally users can mark end of section with `=` repetaed at least once. The `= Section` header will be parsed as a block header, and any steps that follow belong to that section until a new section is defined or the file ends.

### Grammar Changes

In Cooklang grammar, sections would be defined as:
```
section title = { '=' }-, {  text item }, { '=' };
section content = { step }-;
```

Sections are closed either by a new section header or the end of the recipe file.

### Example

Consider the following recipe:

```cooklang
== Dough ==

Mix @flour{200%g} and @water{50%ml} together until smooth.

== Filling ==

Combine @cheese{100%g} and @spinach{50%g} and season to taste.

== Assembly ==
Roll out dough and spread filling on top.
```

This syntax will create three sections, each with a title and content. The Cooklang parser can use this hierarchy to organize recipe steps, which UI applications can display accordingly.


## Effect on applications which use Cooklang

### CookCLI (terminal and web-server)

The addition of sections to Cooklang would improve the functionality of CookCLI, offering improved parsing and output options. With sections, recipes could display a hierarchical structure, making the output more readable and navigable for complex recipes. Key enhancements include:

- Sections can generate a table of contents for easier navigation in lengthy recipes.
- Sections would support recipes that involve independent preparation processes, such as multiple dishes cooked in parallel.
- Ingredients listed under sections improve clarity by associating them directly with the relevant steps.
- Parallel cooking steps can be grouped into sections, aiding workflows that require simultaneous preparation.

### Mobile applications

Mobile applications will need UI updates to accommodate sections. These updates would include:

- Displaying sections in recipe view with clear headers improves readability and provides users with an overview of the recipe’s structure.
- Grouping ingredients by section in recipe view improves user experience when cooking parts of recipe.
- Improved UI handling of sections supports seamless interaction for recipes with separate preparation steps or simultaneous tasks.


## Alternatives considered

1. Using Inline Titles Only. Allow users to simply add titles inline without sectioning syntax. This would avoid grammar changes but wouldn’t provide the parsing structure or UI flexibility that a formalized section syntax can offer.

2. Markdown-Style Sections. Another option was to use a Markdown-style header (e.g., `# Dough`). However, this approach would conflict with existing cookware syntax. Also

3. Component-Based Approach. Sections could have been represented as "components" in separate recipe files. However, "sections" provide a more universally understood term for users, focusing more on organization than on logical "components" or dependencies.


## Acknowledgments

The syntax is inspired by [wiki markup](https://en.wikipedia.org/wiki/Help:Section). The feature has been implemented by [@Zheoni](https://github.com/Zheoni) as a parser extension.
