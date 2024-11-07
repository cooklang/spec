# Note blocks

* Proposal: [0005-note-blocks](0005-note-blocks.md)
* Authors: [Alexey Dubovskoy](https://github.com/dubadub)
* Status: **Awaiting review**

## Introduction

This proposal outlines a plan to add a new block type to the Cooklang language: the note.

This feature will allow recipe authors to embed standalone notes in their recipes,
giving them a way to store relevant background, insights, or personal anecdotes
that aren't part of the cooking steps. Notes won’t be displayed during cooking mode,
but they will appear in the recipe details view, providing context without cluttering
the active instructions.

Discussion thread: ["Additional notes" section which isn't a step, but extra information about a recipe](https://github.com/cooklang/spec/discussions/81)
and [Description, notes, and metadata...](https://github.com/cooklang/spec/discussions/46).

## Motivation

Recipes are more than a sequence of steps; they’re stories, memories, and moments
preserved. Imagine a recipe handed down through generations: a great-grandmother’s
Siberian fish soup, spiced with the unique blend she carried across a journey.
While the specifics of that story might be fictional, the desire to capture memories
like it is very real.

While these stories don’t change the outcome of a dish, they’re significant. Cooklang,
however, currently lacks an effective way to preserve this kind of narrative
information. Metadata or additional steps can somewhat work, but neither
fits naturally, often cluttering the recipe. A dedicated syntax for
notes will let recipe creators embed these details in a way that keeps
them distinct from the active cooking process.

## Proposed solution

The solution is to introduce a new block type in Cooklang for notes, using
a Markdown-inspired syntax for clarity and simplicity.

To add a note, authors will simply use a > symbol at the beginning of a paragraph:

```cook
> This dish is even better the next day, after the flavors have melded overnight.
```

This approach keeps note-taking lightweight and integrates easily with the language,
while visually signaling that the text is extra information. Notes will be displayed
in the recipe details but hidden during cooking, making them easily accessible
when needed without disrupting the cooking flow.


## Detailed design

The design introduces a straightforward syntax addition to the Cooklang grammar.
In a recipe file, any line beginning with `>` will be recognized as a note and
rendered as such in applications that support the new syntax.

Also we should support starting every line with `>`, just like in markdown. For example:

```cook
> This is a long note
> and the user writes the `>` at the beginning of
> every line. All the `>` markup is stripped from the
> note "output".
```

This is optional, so it's the same as:

```cook
> This is a long note
and the user writes the `>` at the beginning of
every line. All the `>` markup is stripped from the
note "output".
```

It's the same as markdown blockquotes but one paragraph per note block.
In Markdown this would be two paragraphs in the same blockquoute but
in Cooklang it's all the same paragraph, newline is removed.

```md
> In markdown
>
> Two paragraphs
```

Unlike Markdown, Cooklang shouldn't support nesting notes inside each other.

### EBNF update

```
recipe = { metadata | step | note }- ;

blank line = { white space }, new line character ;
note marker = ">" ;
note = { note marker, { text item }, blank line }-, blank line ;
```
Symbols and definitions are [here](https://github.com/cooklang/spec/blob/main/EBNF.md).

## Effect on applications which use Cooklang

The addition of note syntax will affect the output of CookCLI. No new sub-commands
are needed; however, the recipe parser must recognize and render > blocks as notes,
showing them in the recipe detail view but hiding them during the cooking view.

### Mobile applications

Mobile apps will need UI adjustments to display notes in the recipe details. Notes
should be visually distinct, perhaps styled in italics or a slightly different color,
ensuring they’re recognized as supplementary information.

## Alternatives considered

### Embedding Notes in Metadata

Storing notes as metadata is a possible approach, but it lacks the intuitive,
in-line placement that allows notes to appear naturally within the recipe flow. Metadata
is typically reserved for more structured data like prep times or dietary information,
so this could lead to cluttered, less-readable recipes.

### Using Special Delimiters in Steps

Another alternative would be to add an indicator within the steps themselves. However,
since notes are not meant to appear in the cooking flow, this would be counterintuitive
and likely disrupt the cooking experience. The block quote syntax keeps notes separate
and accessible without interference.

## Acknowledgments

The feature has been implemented by [@Zheoni](https://github.com/Zheoni) as parser extension.
