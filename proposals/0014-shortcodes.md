# Shortcodes in notes

* Proposal: [0014-shortcodes](0014-shortcodes.md)
* Authors: [Alexey Dubovskoy](https://github.com/dubadub)
* Status: **Awaiting review**

## Introduction

This proposal adds shortcodes to notes. A shortcode is a small tag, `{{< name arguments >}}`, that asks the application to show something richer than text at that point in the recipe. The syntax is borrowed from [Hugo](https://gohugo.io/content-management/shortcodes/). Two shortcodes are standardised here: `image`, for a picture stored next to the recipe or on the web, and `youtube`, for a video. The grammar is generic, so more shortcodes can be added later without changing parsers.

Discussion threads: [Markdown support in steps and notes](https://github.com/cooklang/spec/discussions/138) and [Syntax for associating pictures with steps](https://github.com/cooklang/spec/discussions/58).

## Motivation

Some things in a recipe are easier to show than to describe: what "rough and shaggy" dough looks like, how to pleat a dumpling, when caramel is dark enough. Cooklang has three ways to attach a picture today, all defined in [conventions](../conventions.md#adding-pictures):

- `Baked Potato.jpg` beside `Baked Potato.cook` is the picture of the dish.
- `Baked Potato.3.jpg` is the picture of step 3.
- The `image` metadata key holds a URL to the picture of the dish.

These cover the common case and stay as they are. They leave four gaps:

1. **A step can have one picture.** Showing "before" and "after", or three stages of a fold, isn't possible.
2. **Step pictures are tied to step numbers.** Add a step at the top of the recipe and every picture below it now belongs to the wrong step. Nothing warns about it.
3. **The recipe text doesn't know its pictures exist.** Reading the `.cook` file, you can't tell that a picture is meant to appear, or where.
4. **There is no video.** Authors paste a YouTube link into a note, and applications show it as plain text.

The request has come up more than once. [Discussion 58](https://github.com/cooklang/spec/discussions/58) points out how easily the numbered file names clash and go wrong, and the replies note that people want more than one picture per step. [Discussion 138](https://github.com/cooklang/spec/discussions/138) asks for inline images in notes.

## Proposed solution

Inside a note, `{{< name arguments >}}` is a shortcode. The application replaces it with whatever the name stands for.

```cooklang
---
title: Country sourdough
servings: 8
---

Mix @bread flour{450%g}, @water{340%g} and @sourdough starter{90%g} until no dry flour is left. Rest for ~{30%minutes}.

> The dough should look rough and shaggy at this point.
> {{< image "./shaggy-dough.jpg" alt="Rough, shaggy dough in a glass bowl" >}}

Add @salt{10%g} and fold the dough over itself four times.

> If you haven't done a stretch and fold before, watch one first:
> {{< youtube aBcD3fGh1Jk start=90 >}}
```

An application that supports shortcodes shows the first note as a sentence with the photo under it, and the second as a sentence with a video that starts at 1:30. `./shaggy-dough.jpg` is the file beside the recipe.

An application that doesn't support shortcodes shows the notes as text, tags included. The recipe still opens, and the file name and the video ID are there for the reader to see.

### `image`

```cooklang
> {{< image "./shaggy-dough.jpg" >}}
> {{< image "./steps/fold 2.jpg" alt="Second fold, dough pulled over itself" >}}
> {{< image "https://example.org/pictures/crumb.jpg" alt="Open crumb" >}}
```

The first argument is a path to a local file or a web address. `alt` describes the picture in words.

### `youtube`

```cooklang
> {{< youtube aBcD3fGh1Jk >}}
> {{< youtube "https://www.youtube.com/watch?v=aBcD3fGh1Jk" >}}
> {{< youtube aBcD3fGh1Jk start=90 >}}
```

The first argument is the video ID or a link to the video. `start` is the number of seconds to skip.

## Detailed design

### Where shortcodes are recognised

Shortcodes are recognised **only inside notes**, i.e. blocks that start with `>`. In a step, in a section name and in metadata, `{{< … >}}` stays ordinary text.

Steps are excluded on purpose. In a step, `@`, `#` and `~` start an ingredient, cookware or a timer, and paths and web addresses contain those characters. With the current parser, this step:

```cooklang
See {{< image "https://example.org/@anna/dough~1.jpg" >}} for the texture.
```

adds an ingredient called `anna` and a timer called `1`. Note text has no such syntax, so `@`, `#` and `~` in an argument need no escaping. It also matches what notes are for: a picture or a video is background, not an instruction.

Parsers should warn when a step contains text that would be a valid shortcode in a note, since the author probably meant a note.

### Syntax

A shortcode:

- opens with `{{<` and closes with `>}}`, both on the same line;
- has a **name**: an ASCII letter followed by ASCII letters, digits, `-` or `_`. Names are case-sensitive. Standard names are lowercase;
- has zero or more **arguments** after the name, separated by whitespace.

Whitespace after `{{<` and before `>}}` is optional: `{{<image "a.jpg">}}` is valid.

An argument value is written in one of two ways:

- **Bare**: a run of characters without whitespace or `"`, e.g. `aBcD3fGh1Jk`, `90`, `./steps/fold.jpg`.
- **Quoted**: wrapped in double quotes, e.g. `"./steps/fold 2.jpg"`. A quoted value may contain whitespace.

An argument is **named** when it starts with a key directly followed by `=`, as in `alt="Open crumb"` or `start=90`. A key has the same shape as a shortcode name. There is no whitespace around the `=`. A named value can be empty only when quoted: `alt=""`.

Every other argument is **positional**. This means a bare web address is positional even though it contains `=`: in `https://www.youtube.com/watch?v=aBcD3fGh1Jk` the text before the `=` isn't a valid key.

Positional and named arguments can be mixed. Positional arguments are numbered in the order they appear. If a key is repeated, the last value wins and the parser should warn.

All values are strings. The parser doesn't interpret them.

The shortcode ends at the first `>}}` that is not inside a quoted value. A bare value therefore can't contain `>}}`; quote it if needed.

#### Comments and backslashes

A shortcode is read by the same rules as the rest of the note. [Proposal 0022](https://github.com/cooklang/spec/pull/159) spells out the order for notes: comments are removed first, then the inline content is read, and a backslash makes the next character plain text. Shortcodes follow that order, so a note has one set of rules from start to end.

**Comments are removed first.** Inside a shortcode `--` still starts a comment and removes the rest of the line, the closing `>}}` included. `[- … -]` is removed as well. File names, web addresses and YouTube IDs can contain `--`. Escape the first dash:

```cooklang
> {{< youtube xY\--z012345 >}}
> {{< image "https://example.org/pictures/crumb\--large.jpg" >}}
```

Without the backslash the first line is cut down to `{{< youtube xY`, which is not a shortcode. Parsers should warn when `{{<` and a name have no `>}}` after them on the line. When a comment was removed from that line, the warning should say that `--` inside an argument is written `\--`.

**A backslash makes the next character plain text.** This is how cooklang-rs reads a backslash in a note today: the backslash is dropped and the character after it is kept. Inside a shortcode, an escaped character is always part of a value and never acts as syntax:

| Written | Value |
| --- | --- |
| `xY\--z012345` | `xY--z012345` |
| `"say \"cheese\""` | `say "cheese"` |
| `"a\\b"` | `a\b` |

**Formatting is not read inside a shortcode.** If 0022 is accepted, `*`, `_` and `[` between `{{<` and `>}}` are ordinary characters, so `./my_dough_1.jpg` needs no escaping. A shortcode inside emphasis or link text is still an item of its own; the formatting applies to the text around it.

#### One line only

A shortcode must open and close on the same line of the source file, for two reasons.

A note may span several lines, and the `>` at the start of each line is stripped. A wrapped shortcode whose last line starts with `>}}` would lose its closing tag.

A tag that was cut short must not find its `>}}` on a later line:

```cooklang
> {{< youtube xY--z012345 >}}
> Then shape the loaf. {{< image "./shaping.jpg" >}}
```

The unescaped `--` cuts the first tag. If a shortcode could continue on the next line, it would swallow the sentence and the picture as arguments. With the one-line rule the first line gets a warning and the picture on the second line still works.

#### Text that isn't a valid shortcode

If `{{<` isn't followed by a valid name, valid arguments and `>}}` on the same line, it is not a shortcode. The text stays in the note, and the parser should warn. This covers an unclosed tag, an unterminated quote, and Hugo forms this proposal doesn't adopt:

- closing tags and paired shortcodes: `{{< name >}} … {{< /name >}}`
- the Markdown-rendering delimiters: `{{% name %}}`
- shortcodes nested inside arguments

To write a shortcode as literal text, escape the opening brace. `\{{< image "./a.jpg" >}}` is the text `{{< image "./a.jpg" >}}`, with no warning.

### Parsed model

A note is currently a single string. With this proposal a note is a list of items, in the same way a step is. [Proposal 0022](https://github.com/cooklang/spec/pull/159) makes the same change for formatted text and leaves room for other item types; if both are accepted, `shortcode` is one more item type in that list, next to 0022's text runs.

| Item | Fields |
| --- | --- |
| `text` | `value` |
| `shortcode` | `name`, `positional` (list of strings), `named` (map of string to string), `source` (the shortcode as it appears in the plain text of the note, from `{{<` to `>}}`, with comments removed and escapes resolved) |

For this note:

```cooklang
> The dough should look rough and shaggy at this point.
> {{< image "./shaggy-dough.jpg" alt="Rough, shaggy dough in a glass bowl" >}}
```

the parser returns:

```yaml
- type: text
  value: "The dough should look rough and shaggy at this point. "
- type: shortcode
  name: image
  positional: ["./shaggy-dough.jpg"]
  named:
    alt: "Rough, shaggy dough in a glass bowl"
  source: '{{< image "./shaggy-dough.jpg" alt="Rough, shaggy dough in a glass bowl" >}}'
```

More examples:

| Source | `name` | `positional` | `named` |
| --- | --- | --- | --- |
| `{{< youtube aBcD3fGh1Jk >}}` | `youtube` | `["aBcD3fGh1Jk"]` | `{}` |
| `{{< youtube aBcD3fGh1Jk start=90 >}}` | `youtube` | `["aBcD3fGh1Jk"]` | `{start: "90"}` |
| `{{< youtube https://youtu.be/aBcD3fGh1Jk >}}` | `youtube` | `["https://youtu.be/aBcD3fGh1Jk"]` | `{}` |
| `{{< image src="a b.jpg" >}}` | `image` | `[]` | `{src: "a b.jpg"}` |
| `{{<image "a.jpg">}}` | `image` | `["a.jpg"]` | `{}` |

The parser handles every well-formed shortcode the same way. It doesn't know which names exist, doesn't check arguments, doesn't resolve paths and doesn't touch the network. All of that is left to the application, as it is for [recipe references](0015-other-recipes.md).

Parser libraries should keep offering the note as one string, with shortcodes left in source form. That is what consumers receive today, so code that hasn't been updated keeps working and shows the tag as text.

### Standard shortcodes

#### `image`

| Argument | Required | Meaning |
| --- | --- | --- |
| first positional, or `src` | yes | Path to a local file, or a web address |
| `alt` | no | Description of the picture in words |

**Local files.** This proposal has no path rule of its own. A path is read by the rule that [proposal 0023](https://github.com/cooklang/spec/pull/160) defines for recipe references, so a path means the same thing wherever it is written in a recipe:

| Written | Read from |
| --- | --- |
| `/Shared/knife-grip.jpg` | the root of the recipe collection |
| `./shaggy-dough.jpg` | the folder of the file that contains the note |
| `../Shared/knife-grip.jpg` | the parent of that folder |

The starting point, the handling of `.` and `..` segments, and what the root is are all as in 0023. In particular, a path that climbs above the root resolves to nothing, an application never opens a file outside the collection because a recipe pointed at it, and a leading `/` never means the root of the file system.

Four details are specific to shortcodes:

1. **A path with no prefix is read as if it started with `./`.** `shaggy-dough.jpg` and `./shaggy-dough.jpg` are the same file. In a step the prefix is what tells a reference from an ingredient; in a shortcode argument there is nothing to tell apart.
2. **The file is looked up as written.** No extension is added, and the path is not percent-decoded. Quote a path that contains spaces.
3. **There is no legacy fallback.** 0023 keeps reading an old `./path` reference from the root for a transition period. Shortcodes are new, so there are no old paths to keep working: `./` in a shortcode is only ever the folder of the file.
4. **The separator is `/`.** A backslash is not a separator.

With `Breads/Sourdough.cook`, `Breads/shaggy-dough.jpg`, `Breads/steps/fold.jpg` and `Shared/knife-grip.jpg` in the collection, a note in `Breads/Sourdough.cook` resolves:

| Path | Resolves to |
| --- | --- |
| `shaggy-dough.jpg` | `Breads/shaggy-dough.jpg` |
| `./shaggy-dough.jpg` | `Breads/shaggy-dough.jpg` |
| `./steps/fold.jpg` | `Breads/steps/fold.jpg` |
| `../Shared/knife-grip.jpg` | `Shared/knife-grip.jpg` |
| `/Shared/knife-grip.jpg` | `Shared/knife-grip.jpg` |
| `./Shared/knife-grip.jpg` | nothing (no fallback to the root) |
| `../../knife-grip.jpg` | nothing (above the root) |

This proposal depends on 0023 and should not be released before it. Until then the specification reads `./` from the root, which is the wrong place for a picture that sits beside its recipe.

**Web addresses.** A value that starts with `https://` or `http://` is a web address. Applications must not load any other scheme.

**Formats.** `.jpg`, `.jpeg` and `.png`, as in the pictures convention. Applications may support more.

**Missing pictures.** If the path doesn't resolve or the picture can't be loaded, the application shows the `alt` text, or the path when there is no `alt`. A missing picture is never an error that stops the recipe from opening.

`alt` is read by screen readers and shown when the picture is unavailable. It isn't a visible caption. For a caption, write a sentence in the note next to the shortcode.

#### `youtube`

| Argument | Required | Meaning |
| --- | --- | --- |
| first positional, or `id` | yes | The video ID, or a link to the video |
| `start` | no | Where to start playing, in whole seconds |

Authors copy links more often than IDs, and a link is readable in the plain file, so both are accepted. Applications must recognise these link forms and take the ID from them:

- `https://www.youtube.com/watch?v=ID`
- `https://youtu.be/ID`
- `https://www.youtube.com/shorts/ID`

An ID or a link that contains `--` is written with `\--`, as described under [Comments and backslashes](#comments-and-backslashes).

A value that is neither an ID nor a recognised link is handled like an unknown shortcode.

Applications should show a preview and load the player only when the reader asks for it, and fall back to a plain link when there is no connection.

#### Unknown shortcodes

When an application meets a name it doesn't support, it shows the shortcode's `source` text unchanged. Content never disappears silently, and the result is the same as in an application that has no shortcode support at all.

Applications may add their own shortcodes. To avoid clashes with names standardised later, private names should carry a prefix, e.g. `myapp-gallery`.

### Display

A standard shortcode is shown as a block of its own. The text before it, the picture or video, and the text after it appear one under another. A note that contains only a shortcode shows only the picture or video.

This proposal doesn't change cooking mode. Notes stay out of it ([0005](0005-note-blocks.md)), and so do the shortcodes in them. Showing pictures and video while cooking is left to a separate proposal.

In text-only output, such as a terminal, a plain-text export or a printed recipe without pictures:

| Shortcode | Text form |
| --- | --- |
| `image` | The `alt` text followed by the path or address. Markdown export writes `![alt](src)` |
| `youtube` | `https://www.youtube.com/watch?v=ID`, with `&t=90` when `start` is set |

### Relation to the existing picture conventions

Nothing changes for `Recipe.jpg`, `Recipe.N.jpg` or the `image` metadata key. A picture used in a shortcode is not the picture of the dish and is not a step picture. Applications keep choosing the cover picture the way they do now.

The same file can be used both ways: `Sourdough.2.jpg` is the picture of step 2 by convention, and a note can also show it with `{{< image "./Sourdough.2.jpg" >}}`.

### Remote content and privacy

Loading a picture from the web tells its host that someone opened the recipe, and when. For a recipe you wrote yourself that is your own choice. For a recipe received from someone else it is theirs. Applications that show recipes from other people should consider loading remote pictures and videos only when the reader asks.

Shortcode arguments are data. Applications must not insert them into a page as HTML, and this proposal doesn't define a shortcode for raw HTML.

### Backward compatibility

Existing recipes are not affected unless a note already contains a well-formed `{{< name >}}`. A check of about 16,800 `.cook` files, most of them machine-generated, found no `{{` in any of them.

Recipes that use shortcodes degrade to readable text in older applications. Comments and backslashes work there exactly as they do here, so an older parser shows a tag with its escapes resolved: `{{< youtube xY\--z012345 >}}` appears as `{{< youtube xY--z012345 >}}`. An unescaped `--` cuts the tag in old and new applications alike. The difference is that a new parser warns about it.

Parsers don't change how they read comments or backslashes.

### EBNF update

```
note      = { note marker, { note item }, blank line }-, blank line ;
note item = shortcode | text item ;

shortcode       = "{{<", [ white space ], shortcode name, { white space, shortcode arg }, [ white space ], ">}}" ;
shortcode name  = letter, { letter | digit | "-" | "_" } ;
shortcode arg   = [ shortcode name, "=" ], shortcode value ;
shortcode value = quoted value | bare value ;
quoted value    = '"', { escape | ( text item - '"' - "\" ) }, '"' ;
bare value      = { escape | ( text item - white space - '"' - "\" ) }- ;
escape          = "\", text item ;
```

`letter` and `digit` are ASCII. A `bare value` ends before `>}}`. `escape` is the same rule that 0022 adds for note text. The grammar describes a note line after comments have been removed.

## Effect on applications which use Cooklang

### Specification updates

- `README.md`: a "Shortcodes" subsection under [Notes](../README.md#notes).
- `conventions.md`: a paragraph in "Adding Pictures" that points to the `image` shortcode.
- `EBNF.md`: the grammar above.
- Canonical tests: the suite has no cases for notes yet. Note cases, with and without shortcodes, are added with the specification update, using the `notes` key from [0022](https://github.com/cooklang/spec/pull/159) if that proposal is accepted. Path cases use the file-system-free form described in 0023: the files in the collection, the writing file, the path and the expected result.

### Parsers

Parsers split note text into `text` and `shortcode` items and keep the plain string available. They report a warning for malformed shortcodes in notes and for well-formed ones found in steps. Comment removal doesn't change. Shortcodes have to be read before escapes are resolved into text, or together with them, so that an escaped quote doesn't close a value and an escaped brace doesn't open a shortcode.

### CookCLI (terminal and web-server)

- `cook recipe` prints the text form in the default output. Markdown export writes `![alt](src)` for pictures. JSON and YAML output include the note items. `cooklang` output writes a shortcode back from its name and arguments, quoting values and escaping `"`, `\` and `--` where needed.
- `cook server` shows pictures and videos in the recipe view. Local pictures are served only from inside the recipe collection.
- `cook doctor validate` reports `image` shortcodes that point to a file that doesn't exist, as it already does for missing recipe references.
- No new sub-commands.

### Mobile applications

- The recipe view shows pictures and videos inside notes. Tapping a picture opens it full screen. A video shows a preview and starts playing on tap.
- Cooking mode doesn't change.
- Local pictures have to be on the device, so sync must carry the files that shortcodes refer to, including ones outside the recipe's own folder.
- Without a connection, remote pictures show their `alt` text and videos show a link.

### Editors

Syntax highlighting for the tag, name and arguments. Path completion for `image`, offering files from the current file's folder after `./` and `../` and from the root after `/`, as for references in 0023. A warning for a path that doesn't resolve.

## Alternatives considered

### Markdown image syntax: `![alt](src)`

This is what [discussion 138](https://github.com/cooklang/spec/discussions/138) asks for. It is rejected for three reasons. It covers pictures only: a video needs either a second syntax or guessing from the address what to embed. It has no place for arguments such as a start time. And it differs from a link by one character, which is easy to miss when reading the plain file and easy to get wrong when writing it. [Proposal 0022](https://github.com/cooklang/spec/pull/159) leaves images out of notes for its part, so `![alt](src)` stays a literal `!` followed by a link. Shortcodes give one form for every kind of embed and leave room for more names later without new syntax.

### Embedding bare links automatically

No new syntax at all: an application could turn any picture or video address found in a note into an embed. It can't tell a link the author wanted as a link from one they wanted embedded, there is no place for `alt` or a start time, and a local path such as `fold.jpg` can't be told apart from ordinary words.

### A Cooklang-style sigil: `!image{fold.jpg}`

Consistent with `@`, `#` and `~`, but it spends another sigil character on something that isn't a recipe component, and braces have no natural place for named arguments.

### Pictures and videos in metadata

A `video:` key or a list under `images:` keeps the body clean. It loses the one thing this proposal is for: saying where in the recipe the picture or video belongs.

### More file name conventions

[Discussion 58](https://github.com/cooklang/spec/discussions/58) explores forms like `Chicken French{3}.jpg`. Any file name scheme still depends on step numbers, and still leaves the recipe text unaware of its pictures.

### Shortcodes in steps

Ruled out because of the conflict with `@`, `#` and `~` described above, and because a step is an instruction. If a picture belongs to a step, the step picture convention already handles it.

### A block of its own, without `>`

A line that holds only `{{< image "a.jpg" >}}` could be a new block type. Today that line is a step, so an application without shortcode support would count it as an instruction, number it and show it in cooking mode. Inside a note, the worst outcome is a note with a tag in it.

### A path rule of its own

The proposal could define picture paths itself, for example always from the root, as the specification reads references today. Then every recipe would spell out `Breads/Sourdough/shaggy-dough.jpg` in full and break when its folder is renamed or shared. Any rule that differs from the one for references also gives `./` two meanings in the same file. Taking the rule from [0023](https://github.com/cooklang/spec/pull/160) keeps one meaning for a path wherever it is written, at the price of depending on that proposal.

### The full Hugo feature set

Paired shortcodes with inner content, nesting and `{{% %}}` are powerful in a site generator, where shortcodes are templates. Here they are a fixed set of embeds, and every application has to implement each one by hand. The single self-closing form is enough.

### A generic `video` shortcode

`{{< video "https://…" >}}` would cover every provider and local video files with one name. Each application would then have to work out from the address how to play it. A shortcode named after the provider says exactly what will be embedded. `video` for local files can be added later as its own shortcode.

## Acknowledgments

Thanks to [@pinage404](https://github.com/pinage404) for asking for images in notes, and to [@yb66](https://github.com/yb66) and [@deathau](https://github.com/deathau) for the discussion on step pictures. The syntax comes from [Hugo](https://gohugo.io/content-management/shortcodes/).
