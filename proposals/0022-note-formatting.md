# Inline formatting in notes

* Proposal: [0022-note-formatting](0022-note-formatting.md)
* Authors: [Alexey Dubovskoy](https://github.com/dubadub)
* Status: **Awaiting review**

## Introduction

This proposal allows three kinds of inline Markdown formatting inside [note blocks](0005-note-blocks.md): emphasis (`*text*`), strong emphasis (`**text**`) and links (`[text](destination)`). The parser reads the formatting and returns a note as a list of formatted text runs, so every application shows the same note the same way without needing a Markdown library. Nothing else in a recipe changes: steps, metadata and section names are read as before.

Discussion threads: [Markdown support in steps and notes](https://github.com/cooklang/spec/discussions/138) and [Hyperlinks in Recipe](https://github.com/cooklang/spec/discussions/82).

## Motivation

Notes are the prose part of a recipe: the story behind the dish, a make-ahead tip, a warning, the source it was adapted from. The `>` marker was borrowed from Markdown, and people expect a little more of Markdown to work after it. Bold, italic and links in notes were requested in [#138](https://github.com/cooklang/spec/discussions/138), a link syntax has 15 upvotes in [#82](https://github.com/cooklang/spec/discussions/82), and recipes converted from web pages or Markdown notes arrive with the formatting already in place:

```cooklang
> **Gluten-free option:** see the note below for the flour substitution.

> Optional, but *highly* suggested.

> For the folding technique, watch https://example.com/folding-dumplings
```

The specification treats note text as plain characters, so applications show the asterisks as written, and a web address is only tappable if the application happens to detect it. The author has no way to give a link readable text, or to point from a note to another recipe.

Applications could each run note text through a Markdown library, but that produces different results from one application to the next. A full Markdown renderer turns a note starting with `* ` into a bullet list, a note starting with `#` into a heading, and passes raw HTML through. Each application would also have to pick the same subset and the same edge-case rules by accident.

Defining a small subset in the language, and having the parser read it, gives authors three things they can rely on everywhere: bold, italic and links.

## Proposed solution

Inside a note, three constructs are recognised:

| You write | It means |
| --- | --- |
| `*text*` or `_text_` | emphasis, usually shown in italics |
| `**text**` or `__text__` | strong emphasis, usually shown in bold |
| `[text](destination)` | a link with the given text |

For example:

```cooklang
> **Make ahead:** the dough keeps *up to 3 days* in the fridge.
> See the [shaping guide](https://example.com/shaping) before you start.

Mix @flour{500%g} with @water{350%ml} and @salt{10%g}.
```

The note is displayed as one paragraph: "**Make ahead:** the dough keeps *up to 3 days* in the fridge. See the [shaping guide](https://example.com/shaping) before you start."

A link can point to another recipe with a relative path, which complements [recipe references](0015-other-recipes.md) in steps:

```cooklang
> This filling also works in [Spinach pie](<./Spinach pie.cook>).
```

A backslash makes the next character literal, for the rare case where the text itself contains these characters:

```cooklang
> Use 2\*250 g tins, not 3\*100 g jars.
```

Compared with leaving note text unformatted, formatting that people already type starts to work, and compared with a Markdown library in each application, the result is the same everywhere.

## Detailed design

### Where formatting applies

Formatting is read in note blocks only. Step text, metadata values, section names and the notes attached to an ingredient or cookware (`@flour{200%g}(sifted)`) are unchanged by this proposal.

A note is processed in this order:

1. Comments (`-- …` and `[- … -]`) are removed, as today.
2. The `>` markers are stripped and the lines of the block are joined into one paragraph, as defined in [0005](0005-note-blocks.md).
3. Inline formatting is read on the resulting text. Characters escaped with a backslash count as plain text and never as delimiters.

Because formatting is read after the lines are joined, emphasis and link text may continue from one line of the note to the next. Formatting never crosses from one note block into another.

### Emphasis and strong emphasis

Emphasis and strong emphasis follow the rules of [CommonMark 0.31.2, section 6.2](https://spec.commonmark.org/0.31.2/#emphasis-and-strong-emphasis) without changes. In short:

- A single `*` or `_` on each side marks emphasis; a double `**` or `__` marks strong emphasis.
- An opening delimiter must not be followed by whitespace, and a closing delimiter must not be preceded by whitespace. `a * b * c` has no emphasis.
- `*` works inside a word (`un*believ*able`); `_` does not, so `tomato_sauce_v2.cook` is plain text.
- Formatting can be nested: `**Serve *immediately* or not at all**`, and `***both***` is strong and emphasised.
- A delimiter with no matching partner is ordinary text. `*Use fine sea salt` and `* Day-old rice works best` contain a literal asterisk.

### Links

Links follow the rules for inline links in [CommonMark 0.31.2, section 6.3](https://spec.commonmark.org/0.31.2/#links): link text in square brackets, immediately followed by the destination in parentheses.

- The link text may contain emphasis and strong emphasis, but not another link.
- The destination is either a sequence of characters without spaces (`./dough.cook`, `https://example.com/a_(b)`) or any characters between `<` and `>` (`<./Basic dough.cook>`). A destination with spaces must use the angle-bracket form.
- A title after the destination (`[text](destination "title")`) is accepted, so that links pasted from Markdown still work, and is discarded.
- Square brackets not followed by a parenthesised destination are ordinary text: `[tips]`, `[1]`.

The parser does not resolve, validate or re-encode the destination. It is returned as written, without the angle brackets and with backslash escapes applied.

Only this inline form is recognised. Reference links (`[text][ref]`), autolinks (`<https://example.com>`) and bare addresses are ordinary text; applications remain free to detect bare addresses when displaying a note.

### Backslash escapes

cooklang-rs already reads a backslash in text as an escape: the backslash is dropped and the character after it is kept as plain text, so `\@salt` is the text "@salt" and `\\` is one backslash. This proposal keeps that behaviour and adds one rule: an escaped character never opens or closes emphasis or a link. `\*`, `\_`, `\[` and `\]` are the useful ones in notes.

This is broader than Markdown, where only ASCII punctuation can be escaped. The existing rule is kept so that a backslash means the same thing in a note as in a step. A backslash at the end of a line keeps the behaviour it has today.

### What stays plain text

Everything else that Markdown defines is not part of notes and is returned as ordinary text: code spans, images, raw HTML, character entities such as `&amp;`, Markdown's hard line breaks, lists, headings, nested block quotes, tables and strikethrough. An image `![photo](dough.jpg)` reads as a literal `!` followed by a link.

Cooklang's own markers keep their current meaning inside notes, which is none: `@salt{1%tsp}`, `#pan` and `~{5%minutes}` in a note are plain text.

### Grammar

The note rule in the EBNF becomes:

```ebnf
note        = { note marker, note text, blank line }-, blank line ;
note text   = { strong | emphasis | link | escape | text item } ;

strong      = ( "**", note text, "**" ) | ( "__", note text, "__" ) ;
emphasis    = ( "*", note text, "*" )   | ( "_", note text, "_" ) ;
link        = "[", note text, "]", "(", destination, [ white space, title ], ")" ;
destination = ( "<", { text item - "<" - ">" }, ">" ) | { text item - white space }- ;
title       = ( '"', { text item - '"' }, '"' )
            | ( "'", { text item - "'" }, "'" )
            | ( "(", { text item - ")" }, ")" ) ;
escape      = "\", text item ;
```

The grammar shows the shape of each construct. Which delimiters may open and close emphasis, and how nested or overlapping delimiters are matched, is defined by the CommonMark sections referenced above and cannot be expressed in EBNF.

### Parsed model

Today a parsed note is a single string. With this proposal a note is an ordered list of text runs. Each run has:

| Property | Type | Meaning |
| --- | --- | --- |
| `value` | string | the text of the run, with delimiters and escapes removed |
| `strong` | boolean | the run is inside strong emphasis; omitted when false |
| `emphasis` | boolean | the run is inside emphasis; omitted when false |
| `link` | string | the destination, when the run is inside a link; omitted otherwise |

The note from the example above parses to:

```yaml
- type: text
  value: "Make ahead:"
  strong: true
- type: text
  value: " the dough keeps "
- type: text
  value: "up to 3 days"
  emphasis: true
- type: text
  value: " in the fridge. See the "
- type: text
  value: "shaping guide"
  link: "https://example.com/shaping"
- type: text
  value: " before you start."
```

Rules for the list:

1. A run is the longest stretch of text with the same formatting. Adjacent runs with identical `strong`, `emphasis` and `link` are merged, and empty runs are omitted, so the list for a given note is unique.
2. Nesting is flattened. `**Serve *immediately* or not at all**` gives three runs, all `strong`, the middle one also `emphasis`. A link whose text contains formatting gives several consecutive runs with the same `link`.
3. Concatenating the `value` of every run gives the plain text of the note. Parsers should offer this as a convenience, for consumers that do not display formatting.
4. A note without formatting parses to a single run, whose `value` is the same string that parsers return today.

Runs carry `type: text` like the text items of a step, which leaves room for other item types in notes later without changing the shape.

More examples:

| Note text | Runs |
| --- | --- |
| `Optional, but *highly* suggested.` | "Optional, but " · "highly" *(emphasis)* · " suggested." |
| `***both***` | "both" *(strong, emphasis)* |
| `See [**Basic** dough](<./Basic dough.cook>)` | "See " · "Basic" *(strong, link)* · " dough" *(link)* |
| `*Use fine sea salt` | "\*Use fine sea salt" |
| `Save it as tomato_sauce_v2.cook` | "Save it as tomato_sauce_v2.cook" |
| `This is \*not\* emphasis` | "This is \*not\* emphasis" |
| `**unclosed bold and *closed italic*` | "\*\*unclosed bold and " · "closed italic" *(emphasis)* |
| `[tips] (see below)` | "[tips] (see below)" |
| `Add @salt{1%tsp}` | "Add @salt{1%tsp}" |

### Errors

There are none. Every sequence of characters is a valid note; anything that does not form one of the three constructs is text. Parsers should not warn about unmatched delimiters, because a lone `*` is a common footnote mark.

### Interaction with comments

Comments are removed before formatting is read, and that order has two consequences inside links:

- `--` starts a comment, also in the middle of a destination, and removes the rest of that line. Escape the first dash: `https://example.com/a\--b`. This already applies to addresses in notes and steps today.
- Link text that starts with `-` makes `[-` the start of a block comment, which runs to the next `-]` or, if there is none, to the end of the file. Escape the dash: `[\-5 °C tips](destination)`.

### Scope

This proposal covers note blocks only. Formatting in steps was requested in the same discussions, but steps already carry Cooklang's own inline syntax, and their text is used in cooking mode, timers and read-aloud, so it needs separate consideration. Links in steps remain the subject of the draft proposal [0013 (Hyperlinks)](https://github.com/cooklang/spec/pull/111); if it goes ahead, it should reuse the `[text](destination)` syntax and the `link` property defined here.

Lists, the most requested block-level construct, are discussed under [Alternatives considered](#lists-inside-notes).

### Backward compatibility

No recipe becomes invalid. A note that contains none of `*`, `_` or `[` parses to one run with the same text as before.

Most existing uses of these characters in notes also keep their literal reading:

- A lone footnote asterisk, `*Use fine sea salt`, has no partner and stays literal.
- A bullet written by hand, `* Day-old rice works best`, stays literal because whitespace follows the asterisk.
- Underscores inside a word, `tomato_sauce_v2.cook`, stay literal.
- Square brackets without a parenthesised destination, `[tips]` or `[1]`, stay literal.

A note that already uses Markdown, such as `**Gluten-free option:** …` or `Makes *about* 1 and 1/4 cups filling.`, displays as its author intended for the first time.

Three patterns change their reading without the author meaning it:

- several footnote marks in one paragraph, `(*) … (**) … (***) …`, where the marks pair up as delimiters;
- a name wrapped in underscores, such as a social media handle `@_name_`, where the underscores read as emphasis;
- multiplication written with asterisks, `2*250 g tins or 3*100 g jars`, and any other pair of asterisks that touch the words between them.

In each case a backslash before the character restores the literal reading, and the escaped form already reads correctly in cooklang-rs today.

Parsers and applications that do not implement this proposal keep showing the delimiters as written, which is how such notes look today and remains readable.

## Effect on applications which use Cooklang

### Specification updates

Once accepted, the three constructs and the backslash escape are documented in the Notes section of the language specification and in the EBNF. The canonical tests do not cover notes at present; this proposal adds a `notes` key to the expected result of a test, holding a list of notes, each a list of runs as in [Parsed model](#parsed-model), with tests for the examples in this document.

### Display

1. Show `strong` runs in a heavier weight and `emphasis` runs in italics, relative to however the application styles notes. If notes are already set in italics, emphasis should still be distinguishable, for example by switching back to upright text.
2. Show consecutive runs with the same `link` as one link.
3. Notes stay out of cooking mode, as defined in [0005](0005-note-blocks.md).
4. Where formatting cannot be shown (speech, search indexes, plain-text export, notifications), use the plain text of the note.

### Opening links

A recipe can come from someone else, so a destination is untrusted input. Applications sort a destination into one of three kinds:

| Destination | Kind | What the application does |
| --- | --- | --- |
| starts with `http:`, `https:` or `mailto:` | external | opens it outside the recipe, in the browser or mail client |
| has no scheme and does not start with `//` | path | resolves it like a [recipe reference](0015-other-recipes.md) |
| anything else, or empty | not a link | shows the link text as plain text |

1. Applications must not make a destination of the third kind active. This covers `javascript:`, `data:`, `file:` and every other scheme, as well as protocol-relative addresses (`//example.com`).
2. A path is percent-decoded and resolved the same way as the path of a recipe reference. If it names a recipe in the collection, with or without the `.cook` extension, the application opens that recipe. Applications must not follow a path that resolves outside the recipe collection, whether through `../` or an absolute path, and show the link text as plain text when a path cannot be resolved.
3. Web applications must escape the destination when writing it into a page, and should open external links in a new context without passing the referrer.
4. Applications should let the user see the destination before following an external link, as platforms do with a long press or a hover.

### CookCLI (terminal and web-server)

CookCLI should:

1. In the web UI, render notes with formatting and links, following the rules above.
2. In `human` output, use terminal bold and italic where supported, and print a link as its text followed by the destination in parentheses.
3. In `json` and `yaml` output, keep the existing `type: text` entry for a note with its `value` holding the plain text, and add an `items` list with the runs. Existing consumers keep working and no longer see the delimiters.
4. In `markdown`, `latex` and `typst` output, write the formatting in the syntax of the target format.
5. In `cooklang` output, write the runs back as `**`, `*` and `[text](destination)`, and escape with a backslash any literal character in the text that would otherwise be read as formatting.

No new sub-command is required.

### Mobile applications

Mobile apps show notes as plain text today. They should build the note from its runs as attributed text (`AttributedString` on iOS, `AnnotatedString` on Android), apply bold, italic and link styles, and handle taps on links according to [Opening links](#opening-links). No Markdown library is needed. The recipe detail screen is the only screen affected.

### Editors and syntax highlighting

Editor plugins and grammars (VS Code, Sublime, tree-sitter, CodeMirror) can highlight the three constructs inside notes. Tools that write Cooklang from structured data, such as importers and converters, can keep the bold, italic and links of the source in notes, and must escape literal `*`, `_`, `[`, `]` and `\` where they would otherwise be read as formatting.

### Parsers

Parsers need to read the three constructs in note blocks and change the type of a parsed note from a string to a list of runs. This is a breaking change to the parser's API, although not to the language. cooklang-rs should also expose the plain text of a note, so that consumers that only need the string change one call. Parsers with their own tokenizer can implement the CommonMark delimiter algorithm directly; it is described in the [appendix of the CommonMark spec](https://spec.commonmark.org/0.31.2/#phase-2-inline-structure). Formatting has to be read before escapes are resolved into text, or together with them, so that an escaped `*` is not mistaken for a delimiter. Running a complete Markdown parser over note text is not equivalent, because it would also read lists, headings, code and HTML.

## Alternatives considered

### Leave notes as a string and let applications render Markdown

The parser would be untouched and each application would pass note text to a Markdown library. This is less work in the parser, but each library reads a different dialect, a full renderer interprets far more than the three constructs (a note starting with `* ` becomes a list), and every web application would have to sanitise the resulting HTML itself. Reading the formatting once, in the parser, gives one behaviour and a model that contains nothing but text, two flags and a destination.

### Asterisks only

Recognising only `*` and `**` would leave underscores alone, which avoids surprises with handles and file names, and gives one way to write each thing. It was rejected because Markdown users expect both, `_emphasis_` was the form requested in the discussion, and CommonMark's rule that `_` does not work inside words already protects names like `tomato_sauce_v2`.

### Full CommonMark in notes

Allowing all of Markdown would add lists, headings, code blocks, images, tables and HTML. It makes a note a document inside a document, requires a complete Markdown implementation in every parser, and changes how existing notes read much more often: a note that starts with `* ` or `1.` would become a list.

### Lists inside notes

Lists are the most useful block-level construct for notes ("Variations:" followed by a few bullets), and some authors already write them by hand. They are left out because a note is one paragraph whose lines are joined ([0005](0005-note-blocks.md)); lists require notes to keep their line structure, which changes the model and the display of every existing multi-line note. That deserves its own proposal, and nothing here prevents it: a list item would hold the same runs.

### Formatting in steps as well

See [Scope](#scope). Step text interacts with ingredients, cookware and timers, and is consumed by more features than notes are. Notes are the low-risk place to start, and the same rules can be extended to steps later if there is demand.

### A nested tree instead of flat runs

The parsed note could mirror Markdown's document tree: an emphasis node containing a strong node containing text. A tree distinguishes two adjacent links to the same destination from one link with formatted text, which flat runs cannot. Flat runs were chosen because they match the item lists already used for steps, map directly onto the attributed-text types of mobile platforms, need no recursive types in language bindings, and make the expected result of a test unambiguous.

### One item type per kind of formatting

Each piece could be an item with its own type, `type: emphasis` or `type: link`, next to `type: text`. A single type cannot say that a piece is both strong and emphasised, or that part of a link's text is bold, without nesting items inside items, which is the tree above. Flags on text runs express every combination in a flat list.

### Code spans

Backticks would be easy to add, but recipe notes rarely need them, and leaving the backtick alone keeps it free for other uses in the language.

### Turning bare addresses into links

The parser could emit a link for every `https://…` it finds. It was left out because deciding where an address ends (trailing punctuation, parentheses) is a set of heuristics, not a rule, and applications that want it can already apply the detection their platform provides.

### Keeping link titles

Titles are accepted and discarded. Keeping them would add a property that mobile applications have no natural way to show.

## Acknowledgments

Thanks to [pinage404](https://github.com/pinage404) for the request that started this, to [briansunter](https://github.com/briansunter) for the hyperlinks discussion, and to [Zheoni](https://github.com/Zheoni), who implemented note blocks in cooklang-rs.
