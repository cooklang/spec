# Multiline steps

* Proposal: [0004-multiline-steps](0004-multiline-steps.md)
* Authors: [Alexey Dubovskoy](https://github.com/dubadub)
* Status: **Accepted, in implementation**

## Introduction

The proposal aims to make Cooklang recipes more readable and
git-friendly by treating single line breaks as continuous
text (similar to Markdown), while using double line breaks
to separate distinct recipe steps.

This allows recipe authors to wrap long lines at around
80 characters for better readability in text editors and
cleaner git diffs, without creating unintended step breaks.

It's a simple change that makes recipe writing more natural
while maintaining compatibility with existing applications.

NOTE! This change will break the existing recipes which use single line breaks
to separate steps.

Discussion thread: [Use two line breaks for steps](https://github.com/cooklang/spec/discussions/65) and [Support Bullet Points](https://github.com/cooklang/spec/discussions/60).

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
A step,
next step.

A different step.
```

After:

```cooklang
A step,
the same step.

A different step.
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
