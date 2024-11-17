# Sections

* Proposal: [0006-sections](0006-sections.md)
* Authors: [Alexey Dubovskoy](https://github.com/dubadub)
* Status: **Awaiting review**


Q:
- step images change??
- markdown style syntax important?

## Introduction

This proposal aims to introduce a new block type to the Cooklang language called "section." Sections will serve as structured blocks within recipes, allowing users to group and organize content more intuitively. By breaking recipes into named, clearly defined segments, the Cooklang language can enhance readability, usability, and content structure, especially for more complex recipes.

**Discussion threads**: [Break down recipe instructions further](https://github.com/cooklang/spec/discussions/59), [Functionality for splitting into components and listing component ingredients](https://github.com/cooklang/spec/discussions/72), and [Description, notes, and metadata...](https://github.com/cooklang/spec/discussions/46).

**Extensions link**: [Extensions](https://github.com/cooklang/cooklang-rs/blob/main/extensions.md#section-block)


## Motivation

In its current form, Cooklang organizes recipe instructions and ingredients linearly, limiting the ability to express hierarchical or segmented recipe components. For complex recipes that may involve separate steps (e.g., "Dough," "Filling," "Topping"), or for variations within a recipe (such as optional garnish sections), a flat format can make instructions hard to follow.

Currently, users might work around this by writing instructions with manual section titles, but this is inconsistent and can clutter the recipe file. By adding a formal syntax for sections, we can:

1. Improve readability and maintainability of complex recipes.
2. Allow Cooklang parsers to handle segmented content more effectively.
3. Enhance the user experience across apps by enabling UI components (such as collapsible sections).

In short, the addition of sections offers a formalized way to improve the structure of complex recipes, making them easier to write, read, and implement across Cooklang-compatible applications.


## Proposed solution

We propose to add a new syntax to Cooklang, which introduces sections as explicit blocks. The syntax would look as follows:

```cooklang
= Dough

Mix @flour{200%g} and @water{50%ml} together until smooth.

== Filling ==

Combine @cheese{100%g} and @spinach{50%g} and season to taste.
```

In this example, each `# Section` header represents a new logical group within the recipe. Sections can contain ingredients, instructions, or both, and are delineated with a simple hashtag (=) followed by a section name.

This approach avoids the drawbacks of manually formatted "sections" and offers a consistent and easily parsable syntax for apps and tools using Cooklang.


## Detailed design

The new syntax for sections introduces `=` as a section identifier, followed by the section name. The `= Section` header will be parsed as a block header, and any ingredients or steps that follow belong to that section until a new section is defined or the file ends.

### Grammar Changes

In Cooklang grammar, sections would be defined as:
```
section start = { '=' }-, {  text item }, { '=' }
section
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

For `CookCLI`, this change would enable new parsing and output options. Recipes parsed with sections could display in a hierarchical format, making the CLI output more readable for complex recipes. Additionally, CookCLI might introduce a new sub-command to display sectioned recipes with indentation or collapsible sections.

### Mobile applications

Mobile applications will need to update their UI to support sections, ideally by displaying sections with headers and possibly allowing them to be expanded/collapsed on recipe details screen. Each section should be parsed as a block that the UI can style distinctly, making it easier for users to navigate within a recipe, especially for multi-step or complex recipes.


## Alternatives considered

1. **Using Inline Titles Only**: Allow users to simply add titles inline without sectioning syntax. This would avoid grammar changes but wouldn’t provide the parsing structure or UI flexibility that a formalized section syntax can offer.

2. **Markdown-Style Sections**: Another option was to use a Markdown-style header (e.g., `# Dough`). However, this approach would conflict with existing cookware syntax.

3. **Component-Based Approach**: Sections could have been represented as "components" in the recipe file. However, "sections" provide a more universally understood term for users, focusing more on organization than on logical "components" or dependencies.


## Acknowledgments

The feature has been implemented by [@Zheoni](https://github.com/Zheoni) as parser extension.
