# Canonical metadata

* Proposal: [0007-canonical-metadata](0007-canonical-metadata.md)
* Authors: [Alexey Dubovskoy](https://github.com/dubadub)
* Status: **Released**

## Introduction

Canonical metadata provides a standardised way to attach key information to recipes in Cooklang. This proposal aims to establish a core set of metadata keys, allowing consistent and machine-readable annotations. These enhancements will simplify recipe organisation, sharing, and usability across tools while maintaining Cooklang's minimalist philosophy.

Discussion threads:
- [Add known/standard metadata keys to documentation](https://github.com/cooklang/spec/discussions/101)
- [Description, notes, and metadata...](https://github.com/cooklang/spec/discussions/46)
- ["Additional notes" section which isn't a step, but extra information about a recipe](https://github.com/cooklang/spec/discussions/81)
- [Use metadata for recipe picture URL](https://github.com/cooklang/spec/discussions/64)

Relevant extensions: [Special Metadata](https://github.com/cooklang/cooklang-rs/blob/main/extensions.md#special-metadata)

## Motivation

Recipes are more than just steps and ingredients—they also include context, such as preparation times, authorship, and dietary relevance. While metadata can be approximated through custom fields today, this approach leads to fragmentation and ambiguity. For instance:
- Different users might define preparation time as `prep time`, `time required`, or `duration`.
- There is no convention for structured data, making automation (e.g., scaling servings or sorting by difficulty) more challenging.

This proposal seeks to unify and simplify how metadata is defined and consumed, fostering interoperability between tools and improving the user experience for both developers and end-users.

## Proposed solution

Introduce a standard set of metadata keys with clear purposes and formatting rules. These keys will be optional but encouraged for consistency. Below is the proposed table of metadata:

| Key | Purpose | Example value |
| --- | --- | --- |
| `source`, `source.name` | Where the recipe came from. Usually a URL, can also be text (eg. a book title). | `https://example.org/recipe`, `The Palomar Cookbook <urn:isbn:9781784720995>`, `mums` |
| `author`, `source.author` | The author of the recipe. | `John Doe` |
| `source.url`|The URL of the recipe if nested format is used.|`https://example.org/recipe`|
| `servings`, `serves`, `yield` | Indicates how many people the recipe is for. Used for scaling quantities. Leading number is used for scaling, anything else is ignored but shown as units. | `2`,`15 cups worth` |
| `course`, `category` | Meal category or course. | `dinner` |
| `locale` | The locale of the recipe. Used for spelling/grammar during edits, and for pluralisation of amounts. Uses ISO 639 language code, then optionally an underscore and the ISO 3166 alpha2 "country code" for dialect variants | `es_VE`, `en_GB`, `fr`  |
| `time required`, `time` or `duration` | The preparation + cook time of the recipe. Various formats can be parsed, if in doubt use `HhMm` format to avoid plurals and locales. | `45 minutes`, `1 hour 30 minutes`,`1h30m` |
| `prep time`, `time.prep`|Time for preparation steps only.|`2 hour 30 min`|
| `cook time`, `time.cook`|Time for actual cooking steps.|`10 minutes`|
| `difficulty`|Recipe difficulty level.|`easy`|
| `cuisine`|The cuisine of the recipe.|`French`|
| `diet`|Indicates a dietary restriction or guideline for which this recipe or menu item is suitable, e.g. diabetic, halal etc.|`gluten-free`, or array of values|
| `tags`|List of descriptive tags.|`[2022, baking, summer]`|
| `image`, `images`, `picture`, `pictures`|URL to a recipe image.|`https://example.org/recipe_image.jpg` or array of URLs|
| `title`|Title of the recipe.|`Uzbek Manti`|
| `introduction`, `description`|Additional notes about the recipe.|`This recipe is a traditional Uzbek dish that is made with a variety of vegetables and meat.`|

## Detailed design

The front-matter section in Cooklang files will allow key-value pairs as structured metadata. Keys are case-insensitive. Latest conflicting declaration takes precedence.

## Effect on applications which use Cooklang

### CookCLI (terminal and web-server)

CookCLI might implement a command to list recipes filtered by these standard metadata values. Also the web-server can display canonical metadata with fancy icons.

### Mobile applications

Mobile apps should allow users to search and filter recipes by these canonical metadata values. Also they can display canonical metadata with icons in recipe detail screen.

## Alternatives considered

We try to provide synonyms support, so users can guess most of the time what the value should be.

## Acknowledgments

All discussion thread participants:
- [Add known/standard metadata keys to documentation](https://github.com/cooklang/spec/discussions/101)
- [Description, notes, and metadata...](https://github.com/cooklang/spec/discussions/46)
- ["Additional notes" section which isn't a step, but extra information about a recipe](https://github.com/cooklang/spec/discussions/81)
- [Use metadata for recipe picture URL](https://github.com/cooklang/spec/discussions/64)
