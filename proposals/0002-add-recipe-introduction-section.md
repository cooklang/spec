# Add Recipe Introduction Section

* Proposal: [0002-add-recipe-introduction-section](0002-add-recipe-introduction-section.md)
* Authors: [Chris Born](https://github.com/pelted)
* Review Manager: TBD
* Status: **Awaiting review**

## Introduction

This feature would add support for an introduction section to a recipe. In
addition it would facilitate defining ingredients at the top of a `.cook` file.

Discussion thread: [Description, notes, and
metadata](https://github.com/cooklang/spec/discussions/46).

## Motivation

A title alone is just not enough context to describe a recipe. Often a single
recipe is part of a larger meal plan or tradition. There may be other
side-dishes that pair well with one particular main course; maybe even a wine.
It would be great if we could provide this additional context at the beginning
of a `.cook` file and have it presented in the UI.

Additionally, recipes found online, or from family cookbooks often define
ingredients near the very beginning. *Cooklang* does this already with the way
it parses ingredients and presents that to the user in the UI. This proposal
would also extend this to the creation of a `.cook` file by providing the choice
of pre-defining ingredients in the introduction section.

## Proposed solution

Description text is prefixed with `==` similar to how comments are already
defined in the spec. Ingredients would be prefixed with the comment syntax of
`--` and ignored as steps. (Currently the processor would consider each of these
ingredients as a step unless prefixed as a comment.)

## Detailed design

**Example:**

```cooklang
>> source: https://mexicanfoodjournal.com/easy-salsa-roja/
>> tags: mexican, salsa

== A versatile red salsa prepared with tomatoes that works well with many Mexican dishes.
Also pairs well with the [salsa verde] recipe made with tomatillos.
== Make sure that your tomatoes are really red and ripe which gives the salsa better color and a richer flavor.
Really fresh cilantro gives the salsa a nice herbal tang. Avoid tired wilted cilantro because
it has lost much of its flavor.

-- @roma tomatoes{5}
-- @white onion{1}
-- @serrano chiles{2}
-- @garlic{2%cloves}
-- @cilantro{12%stalks}

Remove the stems, then slice the @serrano chiles{2} in half and remove the seeds. -- Leave the seeds for more heat.

-- remaining steps
```

## Effect on applications which use Cooklang

The biggest change to the applications would be the presentation of the
introduction text in the user interface.

### CookCLI (terminal and web-server)

This proposal would require a change to the CLI output. A description or
introduction section would be expected to be inserted immediately before the
ingredients.

### Mobile applications

The description test would likely require its own initial "tab" to the left of
"Ingredients" This could also be the initial view for a recipe with the
description under the header image followed by the ingredients list. However,
depending on how much text is here it may be a disservice to users to "push
down" list of ingredients.

## Alternatives considered

1. A section break defined by `=====`. Anything before this break indicator is
   considered part of the *introduction/description* section. Text blocks are
   rendered together, and any ingredients defined here are processed as if they
   were in the recipe steps. Anything following this section break would be parsed
   as usual.

2. Description text is prefixed with `==` similar to how comments are already
   defined in the spec. Any ingredients defined alone are considered just that
   and ignored as steps. (Currently the processor would consider each of these
   ingredients as a step.)

After my initial commit of this proposal I evolved a bit in thinking to the
current proposed implementation.

## Acknowledgments

Just wanted to thank the work that has gone into Cooklang to this point. It
really is a great tool and I look forward to helping it along with some new
ideas.
