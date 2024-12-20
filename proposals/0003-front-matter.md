# Front Matter

* Proposal: [0003-front-matter](0003-front-matter.md)
* Authors: [Alexey Dubovskoy](https://github.com/dubadub)
* Status: **Released**
* Pull Request: https://github.com/cooklang/spec/pull/98

* Previous Revision: NA
* Previous Proposal: NA

## Introduction

Idea is to replace the current metadata syntax `>> key: value` with a more
familiar and flexible [YAML](https://yaml.org/spec/1.2.2/) front matter format.

## Motivation

Current metadata syntax is not flexible and very limited in what it can express.
For example, it's impossible to express metadata in a structured way,
like a list of featured ingredients with their quantities and units.

Also, if key contains characters like `:`, it becomes impossible to
parse it using the current syntax.

Instead of investing time and effort in inventing yet another ad-hoc
metadata syntax, it's better to use a well-established and flexible format
like YAML or TOML.

## Proposed solution

Proposal is to replace the current metadata syntax `>> key: value` with a
more flexible [YAML](https://yaml.org/spec/1.2.2/) front matter format. Similar to how
[Markdown](https://en.wikipedia.org/wiki/Markdown) files use front matter to
store metadata at the top of the file (for example, title of the page, date,
etc [Hugo docs](https://gohugo.io/content-management/front-matter/)).

## Detailed design

We should still keep the current `>> key: value` syntax for compatibility
with the existing metadata syntax.

### YAML

[YAML](https://yaml.org/spec/1.2.2/) is a human-readable data serialization standard.

To use YAML front matter, we need to add `---` at the beginning and
`---` at the end of the front matter block.

```yaml
---
title: Spaghetti Carbonara
tags:
  - pasta
  - quick
  - italian
  - comfort food
---
```

## Effect on applications which use Cooklang

Main change is in the syntax of the metadata, so no changes are required
in applications which use Cooklang.

The only problem can be if some applications are using the current metadata
and expect value to be a string. After the change, the value will be not
only a string, but a lists or hashmaps. Also, the value can be a complex
structure with nested lists and hashmaps.

### CookCLI (terminal and web-server)

No changes are required in CookCLI.

### Mobile applications

No changes are required in mobile applications.

## Alternatives considered

Just as a joke, we could use XML or CSV for metadata, but why would anyone
want to use such format for metadata?

Also, alternative is to put effort and develop on current ad-hoc metadata
syntax, but why would anyone want to do that when syntax is familar to everyone
and parsers are already written?

We also discussed an option to support multiple formats at the same time (YAML/TOML/JSON)
but decided to stick with YAML only for simplicity.

## Acknowledgments

Idea is taken from a commentary in [this discussion](https://github.com/cooklang/spec/discussions/63#discussioncomment-8735359).
Thanks to [@ringods](https://github.com/ringods) for the idea.
