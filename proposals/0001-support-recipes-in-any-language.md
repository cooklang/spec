# Support recipes in any language

* Proposal: [0001-support-recipes-not-in-english](0001-support-recipes-not-in-english.md)
* Authors: [Alexey Dubovskoy](https://github.com/dubadub)
* Review Manager: TBD
* Status: **Brainstorming**

## Introduction

Cooklang ecosystem should have unified support of recipes written in any language.

## Motivation

Cooklang gives a way to store recipes in a format that human and machine readable.
Some languages (f.e. German, Finnish, many Slavic languages) would have ingredients
in the middle of sentence to be in a modified form (check an examples on Wikipedia
 https://en.wikipedia.org/wiki/Grammatical_case).

That modified form will lead to creation of multiple entries for the same ingredient
in a shopping list, while it's still the same ingredient and all amounts should be
combined.


## Proposed solution

If config has locale set it should use one instead. In file `./config/main.conf` add line `locale = ru_RU`. Or simple `locale = ru`.

Locale of recipe can be set in file name or metadata. Name file `Koldskål.da.cook`. Or in metadata set

```
>> locale: da_DK
```

- how to singularise ingredients and convert case to base when parsing
- how to singularise units and convert case to base when parsing
- how to pluralise units and ingredients when display with numbers (this can be done by Apple?)

For many languages plural and singular form of an ingredient or unit will be different. To keep recipe readable

https://unicode-org.github.io/cldr-staging/charts/latest/supplemental/language_plural_rules.html
https://cldr.unicode.org/translation/grammatical-inflection

Depending on the locale, there could be up to 6 plural forms used: zero, one, two, few, many, other.

## Detailed design

Describe the design of the solution in detail. If it involves new
syntax in the language, show the additions and changes to the Cooklang
grammar. If it's a new API, show the full API and its documentation
comments detailing what it does. The detail in this section should be
sufficient for someone who is *not* one of the authors to be able to
reasonably implement the feature.

## Effect on applications which use Cooklang

App should guess locale from the system.

### CookCLI (terminal and web-server)

Does the proposal change the output of
[CookCLI](https://github.com/cooklang/CookCLI)? Does it require adding
a new sub-command to fully use this feature?

### Mobile applications

Does the proposal require changes in mobile application UI? Describe
what screens should be changed and how. Give as much details as
possible.

## Alternatives considered

Describe alternative approaches to addressing the same problem, and
why you chose this approach instead.

## Acknowledgments

If significant changes or improvements suggested by members of the 
community were incorporated into the proposal as it developed, take a
moment here to thank them for their contributions. Cooklang evolution is a
collaborative process, and everyone's input should receive recognition!
