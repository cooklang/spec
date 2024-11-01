# Feature name

* Proposal: [0004-multiline-steps](0004-multiline-steps.md)
* Authors: [Alexey Dubovskoy](https://github.com/dubadub)
* Status: **Awaiting review**

## Introduction

A short description of what the feature is. Try to keep it to a
single-paragraph "elevator pitch" so the reader understands what
problem this proposal is addressing.

Discussion thread: [Discussion thread topic for that proposal](https://github.com/cooklang/spec/discussions/65).

## Motivation

Single line breaks are generally used in plain text formats like email
and Markdown to keep paragraphs from displaying as one long line on “dumb”
clients like cat. Two line breaks generally indicate a paragraph break.

Wrapping paragraphs with single line breaks at somewhere
around 80 characters also makes git diffs somewhat nicer to work with.



## Proposed solution

Don’t parse single line breaks as separate steps for the Steps
display in the app. Parse two line breaks as a step break.

## Detailed design

Two examples below show how the same parsed recipe will look before and after the change.

Before:

```cooklang
Crack the @eggs{3} into a blender, then add the @plain flour{125%g}, @milk{250%ml} and @sea salt{1%pinch}, and blitz until smooth.

Pour into a bowl and leave to stand for 15 minutes.
```

After:

```cooklang
Crack the @eggs{3} into a blender, then add the @plain flour{125%g},
@milk{250%ml} and @sea salt{1%pinch}, and blitz until smooth.

Pour into a bowl and leave to stand for 15 minutes.
```

## Effect on applications which use Cooklang

The change is purely internal recipe representation and doesn't
require any changes in the apps. If users don't use double newline between
paragraphs, they will need to update their recipes manually.

## Alternatives considered

Describe alternative approaches to addressing the same problem, and
why you chose this approach instead.

## Acknowledgments

The original idea was suggested in email by Liam Hupfer. It was then implemented
by [@Zheoni](https://github.com/Zheoni) as parser extension.
