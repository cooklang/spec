# Step actions

* Proposal: [0029-step-actions](0029-step-actions.md)
* Authors: [Alexey Dubovskoy](https://github.com/dubadub)
* Review Manager: TBD
* Status: **Awaiting review**

## Introduction

Cooklang marks what a step uses (`@`), what it is done with (`#`) and how long it takes (`~`). It does not mark what the step does: the verb is plain text. This proposal adds one piece of syntax. Text between two backticks in a step is an **action**: `` `Whisk` the @eggs{3}. `` An action may sit anywhere in a step, and a step may hold several.

The proposal has four parts. Part A, the action itself, can be accepted and implemented without the others. Part B says what an application shows. Part C adds an optional action, `` `Garnish?` ``. Part D says which ingredients belong to which action, for applications that draw a table or a graph.

Discussion threads: [Allow verbs in the specification](https://github.com/cooklang/spec/discussions/56), [syntax for tracking combining of ingredients](https://github.com/cooklang/spec/discussions/62), [Syntax for Cooking Methods/Techniques](https://github.com/cooklang/spec/discussions/127), and [cooklang-rs#86](https://github.com/cooklang/cooklang-rs/issues/86).

## Motivation

**Recipes cannot be found by method.** "All braised recipes", "everything I can make without the oven". Discussion #127 and cooklang-rs#86 ask for this; the only way today is to search the step text. The workaround is `tags`, written by hand and apart from the steps they describe.

**A table or a diagram of a recipe has no label for its cells.** A table in the style of Cooking for Engineers, a flow diagram and a step list on a small screen all want two or three words per step: "Fold in", "Bake", "Rest". Today an application guesses them from the sentence, or the author writes them twice.

**A program cannot tell where one action ends and the next begins.**

```cooklang
Melt the @butter{20%g} in a #pan{} and add the @onions{2}.
```

That is two actions in one sentence. In a corpus of about 16,000 recipes, a third of the steps hold two or more actions, and three sentences in ten do not start with the verb ("In a large bowl, whisk…", "Once the oil is hot, add…"). A rule that takes the first word of a step or of a sentence misses those.

The request is as old as the language. Discussion #56 (2021) asked for verbs as a fourth kind of token, with `$` as the sigil. Discussion #62 (2021) asked for a way to track what is combined with what. Discussion #127 (2025) suggested `^boil`.

## Proposed solution

Text between two backticks in a step is an action.

```cooklang
`Melt` the @butter{20%g} in a #pan{} and `add` the @onions{2}.

In a #bowl{}, `whisk` the @eggs{3}. `Fold in` the @flour{100%g}.

`Bake` for ~{25%minutes}. `Garnish?` with @?parsley{}.
```

- The text between the backticks is the action's **name**: `Melt`, `add`, `whisk`, `Fold in`, `Bake`, `Garnish`.
- The span is part of the sentence. With the backticks removed, the step reads as it would have been written without them.
- A `?` before the closing backtick marks the action as optional.
- An application can list a recipe's actions, style them in the step, use them as labels in a table, and search by them.

An application that predates this proposal shows the backticks as text and is otherwise unaffected.

## Detailed design

### Part A: the action

1. **A span is an action.** In a step, the text between two backticks is an action.
2. **Anywhere, several per step.** A span may sit at the start of a step, in the middle or at the end, and a step may hold any number of them.
3. **A span closes on its line.** Both backticks are on the same line of the file. A step written over several lines may hold spans on each line; no span runs across a line break. Backticks pair from left to right along the line: the first with the second, the third with the fourth.
4. **The name is the text between the backticks**, with spaces at both ends removed. It is returned whole. A parser does not split it into words and does not look any word up.
5. **Steps only.** A backtick in a note block, a section name, a metadata value, a comment, or inside the braces or the parentheses of an ingredient, cookware item or timer is text.
6. **An ingredient is never lost.** If the text between two backticks holds an ingredient, a cookware item or a timer, the pair is not an action: the two backticks are text, the component is read as usual, and the parser reports it. `` `Add @salt{}` `` gives the ingredient salt and a warning; the author meant `` `Add` @salt{} ``.
7. **A lone backtick and an empty span are text.** A backtick with no partner on its line stays as written, with no warning. So does a pair with nothing, or only spaces, between. A Markdown code fence line (three backticks, with or without a word after them) is therefore text, as it is today.
8. **Escape.** `` \` `` is a backtick that opens or closes nothing.
9. **Names are compared without case.** `` `Add` `` and `` `add` `` are the same action for search and grouping. The name is kept as written for display.

### Part B: what is shown

10. **A step reads the same with the backticks removed.** An application shows the name in place, without the backticks, and may style it. The author writes the span as part of the sentence: `` `Whisk` the eggs ``, not `` `Whisk` Whisk the eggs ``.
11. **A recipe's actions.** The distinct names, in order of first use, are available to an application for a table, a filter or a search index. This proposal adds no metadata key.

### Part C: optional actions

12. **A `?` directly before the closing backtick marks the action optional.** In `` `Garnish?` with @?parsley{} `` the name is `Garnish` and the action is optional. A name cannot end in `?`.
13. **It says nothing about the ingredients.** An ingredient in the same step is optional only if it carries its own marker, `@?parsley{}` ([0018](0018-optional-components.md)). An application does not need to work out which ingredients belong to which action to build a shopping list.

### Part D: which ingredients belong to which action

Only applications that draw a table or a graph need this. A step view, a shopping list and a search do not.

14. **By position.** The text and the components before the first span belong to the first span. What lies between two spans belongs to the earlier one. What follows the last span belongs to it.
15. **Verb-last languages turn the middle rule round.** When the recipe's `locale` is German, Dutch, Japanese, Korean or Turkish, what lies between two spans belongs to the later one:

    ```cooklang
    @Butter{20%g} in einer #Pfanne{} `schmelzen` und @Zwiebeln{2} `zugeben`.
    ```

16. **A step with no span is one action with no name.** Nothing is guessed from the sentence.

### Grammar

Added to the items a step may hold:

```ebnf
action      = "`", action text, [ "?" ], "`" ;
action text = { text item - "`" - "@" - "#" - "~" - new line } ;
```

The exclusion of `@`, `#` and `~` is wider than rule 6 needs (a `~` followed by a space is not a timer). The exact text follows acceptance.

### What a parser returns

An action is a new item type in a step, with a `name` and, when set, `optional: true`.

```yaml
source: |
  `Melt` @butter{20%g}. `Garnish?` with @?parsley{}.
result:
  steps:
    - - type: action
        name: "Melt"
      - type: text
        value: " "
      - type: ingredient
        name: "butter"
        quantity: 20
        units: "g"
      - type: text
        value: ". "
      - type: action
        name: "Garnish"
        optional: true
      - type: text
        value: " with "
      - type: ingredient
        name: "parsley"
        quantity: "some"
        units: ""
        optional: true
      - type: text
        value: "."
```

### Scope

Not in this proposal:

- **Parameters inside a span**, such as `` `Bake 180 °C` ``, `` `Fry medium-high` `` or `` `Simmer covered` ``. Where the verb ends is known only from a table of verbs (`Stir in`, `Stretch and fold`, `Blind bake 190 °C`), and the specification has no such table. Such a span is valid under this proposal; its name is the whole text, so `Bake 180 °C` and `Bake` are two names. See open question 1.
- **Repeats**, such as `` `Fold ×4` ``.
- **A dictionary form for the verb**, such as `` `Faire revenir:sweat` ``. It waits for the `:` form of [#86](https://github.com/cooklang/spec/pull/86).
- **A table of verbs** with classes or meanings, and any reading of the name by a parser.
- **A metadata key** for techniques.

### Backward compatibility

- **A new file in an application that predates this proposal.** The backticks show as text: `` `Melt` butter ``. Nothing else differs, and no parser fails. Checked with CookCLI 0.38.1 and with cooklang-rs 0.19.0 with every extension on.
- **An existing file in a new application.** A file reads differently only if a line of a step holds two backticks with text between them. No line does in about 16,700 existing recipe files that were checked. Twenty of those files hold a Markdown code fence left by an import, and those lines stay text under rule 7.
- **Markdown viewers** show the span as inline code, which is a fair rendering of a label.

### Relation to other proposals

- **[#159](https://github.com/cooklang/spec/pull/159), inline formatting in notes.** It reads emphasis and links in note blocks and leaves code spans as text. This proposal reads backticks in steps only (rule 5). The two do not meet.
- **[0018](0018-optional-components.md), optional components.** The `?` of an action follows the same idea and is independent of it (rule 13).
- **[#162](https://github.com/cooklang/spec/pull/162), alternatives.** A label at the start of a step (`?frosting:`) comes before the first span.
- **[#165](https://github.com/cooklang/spec/pull/165), intermediate products.** The tail `-> name` at the end of a step belongs to the step's last action.

### Canonical cases

| Source | Reading |
|---|---|
| `` `Whisk` the @eggs{3}. `` | action "Whisk", text, ingredient, text |
| `` In a #bowl{}, `whisk` the @eggs{3}. `` | an action in the middle of the step |
| `` `Cover` and `cook` until golden. `` | two actions |
| `` `Stretch and fold` the dough. `` | one action, "Stretch and fold" |
| `` ` Mix ` `` | action "Mix" |
| `` `Garnish?` with @?parsley{}. `` | action "Garnish", optional; the parsley is optional by its own marker |
| `` `Garnish?` with @parsley{}. `` | the action is optional; the parsley is required |
| `` `Is it done?` `` | action "Is it done", optional |
| `` `Add @salt{}` now. `` | text `` `Add ``, ingredient salt, text `` ` now. ``; reported |
| `` Use the `open one `` | text; not reported |
| `` Use the `open one and `stir` it. `` | action "open one and", then the text `` stir` it. `` |
| ``Empty `` here.`` | text |
| a line of three backticks, or three backticks and a word | text |
| `` \`Not` this. `` | the text `` `Not` this. `` |
| a span opened on one line of a step and closed on the next | text on both lines |
| ``@flour{200%g}(`sifted`)`` | the ingredient's note is `` `sifted` ``; no action |
| `` > Use `00` flour. `` | a note with the text `` Use `00` flour. `` |
| `` = Dough `fast` `` | a section named `` Dough `fast` `` |
| `` title: Soup `v2` `` in the front matter | the title `` Soup `v2` `` |
| `` `Bake 180 °C` `` | action "Bake 180 °C" |
| `` `Добавьте` @соль{} `` | action "Добавьте" |
| `` `ADD` `` and `` `add` `` | two items, equal when compared |

## Effect on applications which use Cooklang

### Specification updates

A section of the README for actions, the grammar above in `EBNF.md`, and canonical tests (about fifteen new, none changed). These follow acceptance.

### Display

The name is shown in place, without the backticks. An application may style it, show a step's actions as a label beside the step, or list a recipe's actions next to its cookware.

### Lists and scaling

Unchanged. An action carries no amount, is on no list and is not scaled.

### Diagnostics

- A pair of backticks that holds an ingredient, a cookware item or a timer (rule 6).

### CookCLI (terminal and web-server)

`cook recipe` prints the name without the backticks. The server styles it. Search can match action names. `cook fmt` keeps spans as written.

### Mobile applications

A new item type in a step. Until an application handles it, the bindings can pass it on as text without the backticks.

### Editors, the language server and syntax highlighting

Colour the span. An editor can pair the backtick and offer a snippet, which matters on keyboards where the backtick is a dead key.

### Converters and other writers

No change is asked of them. A converter that writes spans should write the name only.

### Other parsers

A parser that predates this proposal keeps the backticks as text, so the step stays readable.

## Open questions

1. **Parameters inside a span.** An application that plans or automates cooking wants the heat, the temperature and the lid next to the verb: `` `Bake 180 °C covered` ``. This proposal leaves them out because a parser cannot find where the verb ends without a table of verbs. Two other answers. The specification carries a table of verbs per language, and the span is `verb parameters`. Or the span has a separator, `` `Bake, 180 °C, covered` ``, and no table is needed. With either, rule 10 needs a second look: a sentence that also states the parameter would show it twice ("Rest covered, covered, for 2 minutes").
2. **Display** (rule 10). The alternative is that the span is not part of the sentence: the application hides it in the step and shows it as a label. That frees the author to write any label, and makes a file read badly in every application that predates the proposal.
3. **A component inside a span** (rule 6). The component wins and the pair is text. The alternative, where the span wins, could drop an ingredient from a shopping list with no sign. A third answer reads components inside spans, so that `` `Add @salt{}` `` is an action that contains an ingredient.
4. **A lone backtick** (rule 7) is text and is not reported. It could be reported, at the cost of noise in files with a code fence.
5. **Section names** (rule 5). `` = `Make` the dough `` could label a section. Left out to keep the first version small.
6. **Case** (rule 9). The name is kept as written and compared without case. The parser could instead return it in lower case.
7. **Optional actions** (Part C). The `?` could make the action's ingredients optional too. That needs Part D in every application that builds a list. Or Part C could wait until someone asks for it.
8. **Part D.** The position rule is in the proposal so that two applications draw the same table from the same file. The list of verb-last languages is a first guess and needs readers of those languages. Grouping could instead be left to applications.
9. **Inflected forms.** `` `Whisk` `` and `` `Whisking` ``, or an imperative and an infinitive in languages that have both, are different names. The word-form tables of [#86](https://github.com/cooklang/spec/pull/86) could map them to one; this proposal does not.

## Alternatives considered

1. **A sigil, like the other three tokens:** `$boil` (discussion #56) or `^boil` (discussion #127). It matches `@`, `#` and `~`. Many actions are two or three words (`Fold in`, `Stir together`, `Stretch and fold`), and a multi-word token needs braces: `^fold in{}`. Empty braces after every such verb are noise, and braces invite an amount. `^` has also been suggested for temperature. This is the closest alternative.
2. **Square brackets at the start of a step**, `[Mix]`, as [Gram](https://github.com/gram-lang/gram) does. Brackets already hold block comments (`[- -]`) and are proposed for links; `[Melt](gently)` reads as a Markdown link; and a form tied to the start of a step cannot mark the second action in it.
3. **A metadata list only**, `techniques: [braise, sear]`. It covers search. It has no position in the step, so it gives no labels and no boundaries, and it is written apart from the text it describes.
4. **No markup; find verbs with a word list.** That is guessing, once per language. A fifth of the sentences in the corpus start with "In a", "Once", "When" or the like, and a quarter of the verb-first sentences hold a second action.
5. **Emphasis marks**, `*Whisk*` or `**Whisk**`. [#159](https://github.com/cooklang/spec/pull/159) gives those their Markdown meaning in notes; a second meaning in steps would confuse.
6. **One action per step, always first.** It would split a third of the corpus's steps.
7. **The keyboard.** On US layouts and on phones a backtick is as easy as a bracket. On French, German and Spanish desktop layouts it is a dead key, and Italian layouts have no key for it. A sigil would be easier to type there. Editors can pair and insert the span, and many files are written by converters.

## Acknowledgments

Thanks to the people who asked for this in the discussions linked above, in particular the write-up of a verb-first style guide in discussion #56 and the request for method queries in discussion #127; and to the authors of [Gram](https://github.com/gram-lang/gram), whose step actions showed that a short label per step is enough for a table.
