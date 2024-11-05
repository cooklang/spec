# Canonical metadata

* Proposal: [0007-canonical-metadata](0007-canonical-metadata.md)
* Authors: [Alexey Dubovskoy](https://github.com/dubadub)
* Status: **Awaiting review**

## Introduction

A short description of what the feature is. Try to keep it to a
single-paragraph "elevator pitch" so the reader understands what
problem this proposal is addressing.

Discussion thread: [Add known/standard metadata keys to documentation](https://github.com/cooklang/spec/discussions/101), and [Description, notes, and metadata...](https://github.com/cooklang/spec/discussions/46), ["Additional notes" section which isn't a step, but extra information about a recipe](https://github.com/cooklang/spec/discussions/81) and [Use metadata for recipe picture URL](https://github.com/cooklang/spec/discussions/64).

Extensions link: [Extensions](https://github.com/cooklang/cooklang-rs/blob/main/extensions.md#special-metadata)

## Motivation

Describe the problems that this proposal seeks to address. If the
problem is that some common pattern is currently hard to express, show
how one can currently get a similar effect and describe its
drawbacks. If it's completely new functionality that cannot be
emulated, motivate why this new functionality would help Cooklang community
use Cooklang better.

## Proposed solution

Describe your solution to the problem. Provide examples and describe
how they work. Show how your solution is better than current
workarounds (if any).

## Detailed design

Describe the design of the solution in detail. If it involves new
syntax in the language, show the additions and changes to the Cooklang
grammar. If it's a new API, show the full API and its documentation
comments detailing what it does. The detail in this section should be
sufficient for someone who is *not* one of the authors to be able to
reasonably implement the feature.

## Effect on applications which use Cooklang

Changes in Cooklang syntax often require supplementary UI changes in
apps which use it (otherwise why these changes if nothing will
use them). Describe in details what changes should be implemented.
The detail in this section should be sufficient for someone
who is *not* one of the authors to be able to reasonably implement
the feature.

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
