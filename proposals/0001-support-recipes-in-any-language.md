# Support recipes in any language

* Proposal: [0001-support-recipes-not-in-english](0001-support-recipes-not-in-english.md)
* Authors: [Alexey Dubovskoy](https://github.com/dubadub)
* Review Manager: TBD
* Status: **Awaiting review**

*During the review process, add the following fields as needed:*

* Decision Notes: [Rationale](https://github.com/cooklang/spec/discussions), [Additional Commentary](https://github.com/cooklang/spec/discussions)
* Previous Revision: [1](https://github.com/cooklang/spec/blob/...commit-ID.../proposals/NNNN-filename.md)
* Previous Proposal: [XXXX-filename](XXXX-filename.md)

## Introduction

A short description of what the feature is. Try to keep it to a
single-paragraph "elevator pitch" so the reader understands what
problem this proposal is addressing.

Discussion thread: [Discussion thread topic for that proposal](https://github.com/cooklang/spec/discussions).

## Motivation

Describe the problems that this proposal seeks to address. If the
problem is that some common pattern is currently hard to express, show
how one can currently get a similar effect and describe its
drawbacks. If it's completely new functionality that cannot be
emulated, motivate why this new functionality would help Cooklang community
use Cooklang better.

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
