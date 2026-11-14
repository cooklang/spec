# Alternatives

* Proposal: [0025-alternatives](0025-alternatives.md)
* Authors: [Alexey Dubovskoy](https://github.com/dubadub)
* Review Manager: TBD
* Status: **Awaiting review**

## Introduction

Recipes say "or" all the time: butter or ghee, a blender or a food processor, the oven or the air fryer, home-made pastry or a bought sheet. Cooklang can write this only as the word "or", which no application can read. This proposal adds two pieces of syntax and one metadata key. A `|` between components makes them options for one slot: `@butter{50%g}|@ghee{40%g}`. A label, `?name`, ties an option, a step, a note or a section to a named decision, so that one decision can change several places in a recipe at once. The `choices` key in front matter names those decisions and describes their options.

The proposal has two parts. Part 1, options on a component, can be accepted and implemented without Part 2, labels and `choices`.

Discussion threads: [Syntax for suggesting alternative ingredients](https://github.com/cooklang/spec/discussions/68), [Recipe Variants](https://github.com/cooklang/spec/discussions/75), [Expanding the optional syntax to sections and steps](https://github.com/cooklang/spec/discussions/122)

## Motivation

**The word "or" is all there is.**

```cooklang
Heat the @coconut oil{3%tablespoons} or @extra virgin olive oil{3%tablespoons} in a #saucepan{}.
```

Both oils are on the ingredient list, and the shopping list asks for both. The author's other choice is to leave the second one unmarked ("or olive oil"), and then it is missing from the shopping list, from pantry matching and from search. Cookware has the same problem: `#blender{} or #food processor{}` puts both on the equipment list.

**A choice is often bigger than one ingredient.** There is no form at all for these:

- fresh, active dry or instant yeast, each with its own amount and its own first step;
- the oven or an air fryer: the same ingredients, different steps;
- home-made pastry or a bought sheet: a whole section on one side;
- same-day yeast or overnight sourdough: an extra section, two amounts, steps in three places, and a total time of three and a half hours or twenty-seven.

People handle them with two files that drift apart, or with both versions in prose. Then the ingredient list, the shopping list, the timers and the total time are wrong for at least one version.

**[0018](0018-optional-components.md) left this open.** It made one ingredient or cookware item optional and left steps and sections "for a future proposal".

**Implementations have started without the specification.** [`@tmlmt/cooklang-parser`](https://cooklang-parser.tmlmt.com/v3/guide-extensions.html) has inline alternatives, grouped alternatives and recipe variants in its version 3 pre-release. [Gram](https://github.com/gram-lang/gram), a language derived from Cooklang, writes `@milk{100ml}|@water{95ml}` and `#pan|#wok`. Pull request [#149](https://github.com/cooklang/spec/pull/149) proposes front-matter variables. A recipe written for one of them is not read by the others.

## Proposed solution

Two pieces of syntax:

- **`|` between components** makes them options for one slot. The first is the default.
- **`?label`** ties an option, a step, a note or a section to a named decision. One label can appear in many places, and that is how one decision changes several things at once.

One piece of metadata: **`choices`** in front matter, which names decisions, orders and describes their options, and lets an option replace `time`, `servings` or `yield`.

| Level | What differs between the alternatives | Form |
|---|---|---|
| Component | the name, the amount, or how many components there are | `@fresh yeast{21%g}\|@instant yeast{7%g}`, `#blender{}\|#food processor{}` |
| Step | a step exists on one path only | `?air-fryer: Cook at 200 °C for ~{12%minutes}.` |
| Section | a whole section | `= Pastry ?homemade` and `= Pastry ?shop-bought` |
| Metadata | the time, the servings, the yield; the names and order of the options | `choices:` in front matter |

All four in one file:

```cooklang
---
title: Country Loaf
servings: 8
time: 3 hours 30 minutes
choices:
  leavening:
    same-day: Instant yeast, bake this afternoon
    sourdough:
      description: Levain, bake tomorrow morning
      time: 27 hours
---

= Levain ?sourdough

Mix @active starter{20%g}, @bread flour{50%g} and @water{50%g} in a #jar{}. Leave for ~{8%hours}.

= Dough

Mix @bread flour{500%g}|{450%g}?sourdough, @water{350%g}|{300%g}?sourdough and @fine salt{10%g} in a #large bowl{}.

?same-day: Add @instant yeast{7%g} and knead for ~{10%minutes}. Prove for ~{1%hour}.

?sourdough: Add the levain and squeeze it through the dough. Fold every 30 minutes for ~{4%hours}.

= Bake

Shape into a round and roll it in @mixed seeds{40%g}?seeded.

?sourdough: Rest in the fridge for ~{12%hours}.

Bake in a #dutch oven{}|#baking stone{}[with a tray of water under it for steam] at 240 °C for ~{40%minutes}.
```

An application shows one picker, "Leavening: same-day · sourdough", one switch, "Seeded", and "dutch oven or baking stone" in the equipment. What each path holds:

| | Default (`same-day`) | `sourdough` |
|---|---|---|
| Sections | Dough, Bake | Levain, Dough, Bake |
| Bread flour | 500 g | 50 g + 450 g |
| Water | 350 g | 50 g + 300 g |
| Leavening | instant yeast 7 g | active starter 20 g |
| Mixed seeds | not used unless "Seeded" is on | the same |
| Time | 3 h 30 | 27 h |

## Detailed design

### Part 1: options on a component

A `|` after an ingredient or a cookware item, followed by another option, makes an **option group**. A group is one slot in the recipe: exactly one of its options is used.

An option takes one of four forms:

| Form | Example | Meaning |
|---|---|---|
| A component | `@fresh yeast{21%g}\|@instant yeast{7%g}` | its own name and its own amount |
| A component with no amount | `@haddock{500%g}\|@cod\|@pollock`, `@double cream{300%ml}\|@crème fraîche{}` | takes the first option's amount |
| An amount with no name | `@water{350%g}\|{300%g}?sourdough` | takes the first option's name |
| Several components in round brackets | `@egg{1}\|(@ground flax{1%tbsp} + @water{3%tbsp})` | they stand in for one together |

Rules:

1. **Where `|` counts.** Only between two options: after a complete option and before `@`, `#`, `{` or `(`. Spaces, and one line break inside a step, are allowed on either side: `@parsley{} | @mint{}`. Anywhere else a `|` is text, as in `180C|355F`. A pipe inside the braces of a component is not touched.
2. **The sigil is repeated** on every option that has a name: `@butter{}|@ghee{}`, not `@butter{}|ghee{}`. Each option is then a complete component, so a parser that knows nothing of this proposal still highlights and lists it.
3. **Order.** The first option is the default. Parsers and writers keep the order.
4. **Taking the amount from the first option.** A later option written with no braces (one-word names only) or with empty braces takes the first option's amount, both the quantity and the unit, and scales as the first option does: in `@haddock{500%g}|@cod`, cod is 500 g. A name of more than one word needs its braces, as everywhere in Cooklang: `@crème fraîche{}`. When the first option has no amount, or is a bracketed option, there is nothing to take. The author is saying the swap is one for one; when it is not, each option gets its own amount.
5. **Taking the name from the first option.** `|{300%g}` is the first option's ingredient in another amount. It needs a label (Part 2), because nothing else says when to use it.
6. **One for many.** Round brackets hold two or more components joined by `+`, with optional spaces around the `+` and nothing else inside. Brackets are an option only when a `|` touches them, so `(@salt{} and @pepper{})` in prose stays text with two ingredients. A bracketed option may come first: `(@lemon juice{30%ml} + @sugar syrup{15%ml})|@sour mix{45%ml}`.
7. **What follows an option**, in this order: its braces, its `(preparation)`, its `?label` (Part 2), its `[reason]`. After a bracketed option the label and the reason follow the closing bracket.
8. **Reason.** `[text]` after an option says when to pick it: `@galangal{2%cm}|@ginger{2%cm}[if galangal is unavailable]`. It belongs to the option it follows. It is display text and is never parsed. It may wrap across a line inside a step. It is read only after an option of a group or after a labelled component; anywhere else square brackets stay text. `[-` still opens a block comment.
9. **Kinds.** Ingredients, cookware and recipe references can be options. A group may mix an ingredient and a cookware item: `#pie weights{}|@dried beans{}`. Timers cannot be options; a time that differs belongs in a labelled step.
10. **No nesting.** A bracketed option holds components, not groups.
11. **Optional.** The `?` marker of [0018](0018-optional-components.md) on the first option makes the slot optional: `@?sour cream{}|@yoghurt{}` is "optionally, sour cream or yoghurt". It is accepted on later options and changes nothing there. Otherwise, switching from sour cream to yoghurt would turn an optional topping into a required one.

In a group with no labels, the first option is the one the recipe uses, and the others are what the cook may use in its place.

**If Part 1 is accepted alone,** two things wait for Part 2: the amount with no name (rule 5) and the label in rule 7. Everything else in Part 1 stands without labels.

### Part 2: labels

A **label** is a `?` followed directly by a name: `?sourdough`, `?air-fryer`, `?yeast=fresh`.

- **Characters.** Letters and digits of any script, and hyphens after the first character. No spaces. Labels are compared without regard to case.
- **Qualified form.** `?choice=option` names the decision as well as the option: `?yeast=fresh`. Both sides follow the character rule, and there is one `=`.
- **Where a label ends.** At the first character that is neither a label character nor the one `=` of the qualified form: `@chilli flakes{1%tsp}?spicy.` is the label `spicy` and a full stop.
- **Where a label is read.** In three places only, and only when a label character follows the `?` directly:
  1. at the very start of a step or a note, followed directly by a colon: `?air-fryer: Cook at …`;
  2. as the last word of a section line: `= Pastry ?homemade`;
  3. directly after an ingredient, a cookware item, a timer or an option: `@oat milk{}?dairy-free`, `~{12%hours}?sourdough`.

  Everywhere else `?` is text. `Got a #grill{}? Fire it up.` and `Add @salt{}?` are unchanged, because a space or the end of the line follows the `?`.

**On an option.** A labelled option is used when its label is active: `@whole milk{300%ml}|@oat milk{}?dairy-free` is oat milk with "dairy-free" on and whole milk otherwise.

**On a single component.** `@chilli flakes{1%tsp}?spicy` is in the recipe only when "spicy" is on. `@?chilli flakes{1%tsp}` from [0018](0018-optional-components.md) stays what it is: one ingredient the cook may skip, with no name and no link to anything else. A label is a named switch that can move several things at once.

**On a timer.** `Rest in the fridge for ~{12%hours}?sourdough` has that timer only when "sourdough" is active. A timer takes a label but is never an option: a `|` between two timers is text, so `~{40%minutes}|~{12%minutes}?air-fryer` is two timers, the second with a label.

**On a step.** A step that starts with `?label:` is in the recipe only when the label is active. The label covers the whole step, however many lines it has, and everything in it: its ingredients, its cookware and its timers. The prefix is not part of the step's text. A space after the colon is optional.

**On a note.** The same prefix after the note marker: `> ?sourdough: Feed the starter the night before.` Otherwise a note about one path shows on every path.

**On a section.** `= Pastry ?homemade` and `== Pastry ?homemade ==` are the section "Pastry", in the recipe only when "homemade" is active. The label is not part of the name. It covers every step and note up to the next section line. A line holding only `=` already starts an unnamed section, and that is how a labelled section is closed before steps that every path shares. Two rules follow:

- Two sections may carry the same name when their labels are options of one choice. They are alternatives for that part of the recipe.
- A step inside a labelled section may carry its own label. It is in the recipe when both labels are active. The same holds for a labelled component inside a labelled step.

**One label per place.** There is no "and", "or" or "not". A step for the opposite of a toggle is written by making a choice with two options: `?milk=dairy:` and `?milk=oat:`.

### Choices and toggles

A **choice** is a named decision with two or more options, of which exactly one is active. A **toggle** is a single label that is on or off. Which of the two a label is gets declared. It is never inferred from where the label sits:

1. **Declared in front matter:** `choices:` lists choices and their options in order. The first option is the default.
2. **Declared in the text:** a qualified label, `?yeast=fresh`, adds the option `fresh` to the choice `yeast`. Options are ordered by first appearance and the first is the default. Where front matter also declares the choice, its order wins.
3. **A bare label that is an option of a declared choice** means that option, anywhere in the file. That is how one decision reaches a section, three steps and two amounts.
4. **Any other bare label** is a toggle, off by default. Front matter can turn its default on.

A bare label that is an option of two different choices (`?crust=plain` and `?filling=plain`, then `?plain`) is an error; it has to be qualified.

The **default path** is every choice at its first option and every toggle at its default. A **selection** is a set of options and toggle states that differs from it. A label is **active** when the selection, or else the default path, has its option chosen or its toggle on.

The **active option** of a group is the first option whose label is active; if there is none, the first option with no label; if there is none of those either, the slot is empty on that path. When the labels of two options of one group are active together, the first in the text is used.

### The `choices` key

```yaml
time: 3 hours 30 minutes
choices:
  leavening:                                  # a choice; options in order, the first is the default
    same-day: Instant yeast, bake this afternoon     # option: description
    sourdough:                                       # option: description and overrides
      description: Levain, bake tomorrow morning
      time: 27 hours
  seeded: {toggle: true, description: Rolled in mixed seeds}   # a toggle
```

- A choice is a mapping from option name to either a description or a mapping.
- An option's mapping may hold `description` and overrides for `time` (in any of its [canonical spellings](../conventions.md#canonical-metadata): `time required`, `duration`, `prep time`, `cook time`, `time.prep`, `time.cook`), `servings` and `yield`. Any other key is kept and has no effect.
- A toggle is a mapping with `toggle: true`. It may hold `default: on` to start switched on (`true` is read the same), and also a `description` and the same overrides, which apply when it is on. A toggle that needs none of these does not have to be declared.
- `toggle` is a reserved word in `choices` and cannot be the name of an option.
- `choices` is optional. A file whose choices are declared in the text and need no description has no reason to write it.

Rules for the rest of the front matter:

1. **Front matter describes the default path.** A reader that ignores choices shows a recipe whose title, time, servings and tags are all true of what it shows.
2. **An override replaces a value whole** while its option is active. There are no deltas and no arithmetic. When two active options override the same key, the option of the choice declared later wins.
3. **`diet`, allergens and nutrition cannot be overridden.** Values in front matter are claims about the default path. See [Diet, allergens and nutrition](#diet-allergens-and-nutrition).

### Grammar

The [EBNF](../EBNF.md) does not yet hold sections or preparations, so the rules below add what this proposal needs and leave the rest as it is.

```ebnf
(* Part 1 *)
slot             = option, { gap, "|", gap, option } ;
option           = ( component option | amount option | bracketed option ), [ label ], [ reason ] ;
component option = ( ingredient | cookware ), [ preparation ] ;
amount option    = "{", amount, "}" ;
bracketed option = "(", { white space }, component option,
                   { { white space }, "+", { white space }, component option }-,
                   { white space }, ")" ;
preparation      = "(", { text item - ")" }, ")" ;
reason           = "[", { text item - "]" }-, "]" ;
gap              = { white space }, [ new line character, { white space } ] ;

(* a one-word name also ends at "|" *)
word             = { text item - white space - punctuation character - "|" }- ;

(* Part 2 *)
label            = "?", label name, [ "=", label name ] ;
label name       = label character, { label character | "-" } ;
label character  = ? Unicode General Category L* or Nd ? ;

labelled timer   = timer, label, [ reason ] ;
labelled step    = label, ":", { white space }, step ;
labelled note    = note marker, { white space }, label, ":", { white space }, { text item } ;
section line     = "=", { "=" }, { white space }, [ section name ], [ white space, label ], { white space }, { "=" } ;
```

Notes on the grammar:

- A `slot` with one option is an ordinary component. With two or more it is a group.
- An `amount option` cannot be the first option. A `bracketed option` can.
- A timer is never part of a `slot`. It can only carry a label.
- A `reason` is read only when the slot is a group or the option has a label. Inside a step it may hold one line break, which `text item` does not show.
- `|` is in Unicode category Sm, not P, so by today's `word` rule a one-word name would run through it. The `word` rule above ends the name there.
- A `gap` allows one line break on each side of the `|`, inside a step.

### Scope

Not in this proposal:

- **How a selection is written outside the recipe:** in a `.shopping-list` file, in a `.menu` file, or on a reference from another recipe. Until that is defined, a reference with no selection means the default path. See [Open questions](#open-questions).
- **Expressions over labels** ("and", "or", "not").
- **Timers as options.** A timer can carry a label, but a group holds ingredients and cookware only. See [Open questions](#open-questions).
- **Conversion or substitution ratios.** The author writes each option's amount.
- **A substitution that holds across recipes**, such as "this household uses oil for butter". A group is the swap the recipe's author wrote down.
- **A version that changes most of the recipe.** That is a second file.

### Backward compatibility

**Existing files.** Four forms change their reading:

| Written | Today | Proposed |
|---|---|---|
| `@a{}\|@b{}`, and the other option forms of Part 1 after a `\|` | components with a `\|` in the text between them | one group |
| `@a{}?word`, `#a{}?word`, `~{…}?word` | a component or a timer, and the text `?word` | a component or a timer with a label |
| `?word: …` at the start of a step or a note | text | a labelled step or note |
| `= Name ?word` | a section named "Name ?word" | the section "Name" with a label |

No file with any of these forms outside a comment was found in the recipe collections checked while writing this proposal. Forms that are close stay as they are:

| Written | Reading, today and proposed |
|---|---|
| `at 180C\|355F` | text: there is no option on either side of the pipe |
| `{2%cups\|260%g}` | the pipe is inside braces ([#157](https://github.com/cooklang/spec/pull/157)) |
| `@white wine\|wine{100%ml}` | the pipe is inside a name, where the [cooklang-rs alias extension](https://github.com/cooklang/cooklang-rs/blob/main/extensions.md) reads it |
| `Got a #grill{}? Fire it up.`, `Add @salt{}?` | text: a space or the end of the line follows the `?` |
| `@urad dal{150%grams}[black lentils]` | text: there is no group and no label in front of the brackets |
| `= Is it done?` | a section named "Is it done?" |

[0018](0018-optional-components.md) turned down a suffix marker, `@thyme{}?`, because `Did you add the @salt{}?` is an ordinary sentence. A label is not that marker: it needs a name glued to the `?`. That sentence stays text.

**New files in applications that predate this proposal.** Checked with CookCLI 0.38.1:

| Written | The step reads | The lists hold |
|---|---|---|
| `@whole milk{300%ml}\|@oat milk{}?dairy-free` | "whole milk\|oat milk?dairy-free" | whole milk 300 ml, and oat milk with no amount |
| `@haddock{500%g}\|@cod\|@pollock` | "haddock\|cod\|pollock" | all three; cod and pollock with no amount |
| `@water{350%g}\|{300%g}?sourdough` | "water\|{300%g}?sourdough" | water 350 g, which is right for the default path |
| `@egg{1}\|(@ground flax{1%tbsp} + @water{3%tbsp})?vegan` | "egg\|(ground flax + water)?vegan" | all three |
| `?yeast=fresh: Dissolve @fresh yeast{21%g} …` | the label as the step's first word | every yeast step is shown, and every yeast is on the list |
| `= Pastry ?homemade` | a section titled "Pastry ?homemade" | both pastry sections are shown and counted |
| `~{12%hours}?sourdough` | "12 hours?sourdough" | nothing; the timer is there on every path |

Nothing is reported as an error, and the text stays readable: it looks like a cookbook that prints every variant. The lists hold every option of every choice. For "or" between two marked components that is what happens today. For steps and sections it is new: a file with three yeasts buys three yeasts, and the Country Loaf above lists 550 g of flour and 400 g of water, which is neither path. This is why [applications that write recipes](#converters-and-other-writers) should wait for the applications that read them.

**Parsers that follow the EBNF to the letter** may read a one-word option straight before a pipe with the pipe in its name (`@cod|@pollock` as an ingredient named `cod|`), until they adopt the `word` rule above.

### Relation to other proposals

| Proposal | How they meet |
|---|---|
| [0018](0018-optional-components.md), optional ingredients and cookware (accepted) | Unchanged. `@?` is the unnamed optional; a toggle is the named one. Rule 11 of Part 1 says what `@?` does in a group. Its selection lines in `.shopping-list` files are not touched |
| [#157](https://github.com/cooklang/spec/pull/157), alternative units | No clash: its pipe is inside the braces, this one is between components |
| [#149](https://github.com/cooklang/spec/pull/149), component variables | A different route to ingredient choices: the options are listed in front matter under a variable name, and the step writes the variable, `@yeast{4%g}`. Here the options sit in the step where the cook reads them, and a choice used in several steps is tied together by a label. Its split of one total amount between steps and its cookware identities are not covered by this proposal |
| [#161](https://github.com/cooklang/spec/pull/161), optional `%` | An amount in an option is read by the same rule as any other amount |
| [#160](https://github.com/cooklang/spec/pull/160), reference paths | A recipe reference in an option is resolved by the same rule as any other reference |
| [#156](https://github.com/cooklang/spec/pull/156), hidden components | Each option is a complete component, so the hidden marker applies to the option it is written on |
| [#159](https://github.com/cooklang/spec/pull/159) and [#112](https://github.com/cooklang/spec/pull/112), formatting and shortcodes in notes | A note's label prefix comes first; formatting and shortcodes are read in what follows it |

### Canonical cases

| Source | Reading |
|---|---|
| `@fresh yeast{21%g}\|@instant yeast{7%g}` | one group; default fresh yeast 21 g; other option instant yeast 7 g |
| `@haddock{500%g}\|@cod\|@pollock` | three options; cod and pollock take 500 g |
| `@double cream{300%ml}\|@crème fraîche{}` | crème fraîche takes 300 ml |
| `@double cream{300%ml}\|@crème fraîche and stir` | the second option is `crème`, with 300 ml; "fraîche and stir" is text |
| `@parsley{} \| @mint{}` | one group; spaces around the pipe |
| `@water{350%g}\|{300%g}?sourdough` | water 350 g; 300 g when `sourdough` is active |
| `@egg{1}\|(@ground flax{1%tbsp} + @water{3%tbsp})?vegan` | egg by default; flax and water together when `vegan` is on |
| `(@lemon juice{30%ml} + @sugar syrup{15%ml})\|@sour mix{45%ml}` | a bracketed option first |
| `Season (@salt{} and @pepper{}) well` | no group; two ingredients |
| `#stand mixer{}\|#electric whisk{}` | a cookware group |
| `#pie weights{}\|@dried beans{}` | a mixed group; the default is the cookware |
| `@?sour cream{}\|@yoghurt{}` | an optional slot with two options |
| `@butter{40%g}(soft)\|@olive oil{3%tbsp}?dairy-free[if no butter]` | a preparation on the first option; a label and a reason on the second |
| `@white wine\|wine{100%ml}` | no group: the pipe is inside a name |
| `at 180C\|355F` | text |
| `~{10%minutes}\|~{6%minutes}` | two timers and a text pipe |
| `~{12%hours}?sourdough` | a timer, in the recipe when `sourdough` is active |
| `~{40%minutes}\|~{12%minutes}?air-fryer` | two timers and a text pipe; the second timer has the label `air-fryer` |
| `@urad dal{150%grams}[black lentils]` | no group and no label: the square brackets are text |
| `@chilli flakes{1%tsp}?spicy` | in the recipe when the toggle `spicy` is on |
| `Got a #grill{}? Fire it up.` | text |
| `Add @salt{}?` | text |
| `?yeast=fresh: Dissolve @fresh yeast{21%g}.` | a step for the option `fresh` of the choice `yeast` |
| `? Maybe: add more.` | text: a space follows the `?` |
| `?maybe add more.` | text: no colon follows the label |
| `> ?sourdough: Feed the starter the night before.` | a note for `sourdough` |
| `= Pastry ?homemade`, `== Pastry ?homemade ==` | the section "Pastry" with the label `homemade` |
| `= Is it done?` | a section named "Is it done?" |
| `?fresh:` after `?yeast=fresh:` | the same option |
| `?plain:` when `crust=plain` and `filling=plain` are declared | an error: the label is ambiguous |
| `?garnish:` never declared | a toggle, off by default |

## Effect on applications which use Cooklang

An application that does not know which path the reader wants works on the default path.

### Specification updates

Once accepted: sections for options and labels in the README, the grammar rules in the EBNF, `choices` in the canonical metadata table of the conventions, and canonical tests.

### Display

1. One picker per choice and one switch per toggle, with the defaults applied. The choice name is the heading and the descriptions are the picker's text. Without a description, a hyphen in a label is shown as a space.
2. Steps, notes and sections that are not on the active path are not shown. Two sections with the same name and labels of one choice are shown as one heading.
3. A group is shown as one line in the ingredient and equipment lists: the active option, with the other unlabelled options named beside it ("haddock 500 g, or cod, or pollock"). An application may let the reader switch between the unlabelled options of a group. Options with labels are switched by their label.
4. A reason is shown with its option.
5. A "show all" view prints every option with its label, the way a cookbook prints "if using an air fryer". This is also what a plain-text reader sees.
6. Listings and search may summarise across paths, for example the time as a range, "3 h 30 to 27 h", taken from the default value and the overrides.
7. Exports that have no way to carry a choice (schema.org, print, plain text) use the default path and write the other options as text.

### Lists, scaling and totals

1. The ingredient list, the equipment list, the timers and the total time follow the active path. Components and timers that are off, or that sit in steps and sections that are off, are not counted.
2. Each option scales by its own amount. An amount taken from the first option scales with it.
3. Only active options add to a total. The rule of 0018 that optional amounts are listed apart from required ones still holds.

### Shopping lists

The active option only, with its amount. The other unlabelled options should be named on the line, because that choice is often made in the shop.

### Pantry matching

A slot is satisfied when its active option or any of its other unlabelled options is in stock.

### Diet, allergens and nutrition

`diet` and other claims in front matter are about the default path. An application may show a claim about another path ("vegetarian option", "dairy-free option") only when it has computed the claim from that path's ingredients, never from a tag or a label name. A recipe listed as dairy-free because one of its paths is dairy-free can hurt someone.

### Diagnostics

Tools that check recipes should report:

- a bare label that is an option of two choices (an error);
- a label declared in front matter and never used;
- a toggle whose name is one edit away from a declared option (`?sourdouhg`);
- a choice with one option;
- an amount with no name (`|{300%g}`) and no label;
- a `|` between two timers, which is text and not a group;
- two options of one group whose labels can be active together;
- two active options that override the same metadata key;
- a key in an option's mapping that is not a description or a known override.

### CookCLI (terminal and web-server)

1. `cook recipe` and `cook shopping-list` work on the default path, with a way to pass a selection.
2. `cook doctor` gets the diagnostics above.
3. The web UI shows the pickers and switches, and "or …" on group lines.

### Mobile applications

1. A parser update.
2. A picker row per choice and a switch per toggle above the ingredients.
3. "or …" on the lines of a group in the ingredient list, the equipment list and the shopping list.
4. Cooking mode skips steps that are off.

### Editors, the language server and syntax highlighting

1. Highlight the pipe between options, labels and reasons.
2. Complete declared labels after `?`.
3. Show the diagnostics above.

### Converters and other writers

Applications that create recipes (importers, converters, assistants) should not write groups or labels until the applications people read recipes in have shipped support. An application that predates this proposal puts every option of every choice on the shopping list.

Writers keep the order of options.

### Other parsers

Parsers that are not built on cooklang-rs follow the canonical tests. Until they do, they read the new forms as shown under [Backward compatibility](#backward-compatibility).

## Open questions

1. **A selection outside the recipe.** A shopping list, a menu and a reference from another recipe all need a way to say "the sourdough path of Country Loaf". This proposal does not define one. The selection lines that 0018 added to `.shopping-list` files are the nearest precedent.
2. **A name with no braces straight before a label or an amount option.** By the rules above `@cod?mild` is the ingredient `cod` with the label `mild`. Requiring braces before a label (`@cod{}?mild`) would be stricter and would leave `@cod?mild` as text. In the same way `@water|{300%g}` can be read as water followed by an amount option, where today the name runs to the brace and the ingredient is called `water|`. Written with its braces, `@water{}|{300%g}`, it reads one way only. A named timer with no braces before a label, `~rest?slow`, is the same case as `@cod?mild`.
3. **Timers as options.** A time that differs between two paths is written as a labelled step, or as two labelled timers. `~{40%minutes}|~{12%minutes}?air-fryer` would be shorter and would match `@water{350%g}|{300%g}?sourdough`, but as proposed it is two timers with a text pipe between them. Whether a group should hold timers is open.
4. **A labelled section with no name.** `= ?sourdough` reads as an unnamed section with a label. It could be made an error.
5. **The opposite of a toggle.** There is no "not". A step that is in the recipe only when a toggle is off needs a choice with two options. Whether that is enough is a question for people who write such recipes.
6. **Canonical tests.** The tests need a shape for a group, for a label and for `choices`. This proposal defines the syntax and its meaning, not the parsed model.

## Alternatives considered

1. **The word "or" and nothing else.** It is what files hold today. No application can read it, and it needs translating.
2. **The sigil written once:** `@milk{200%ml}|almond milk{100%ml}`, as in `@tmlmt/cooklang-parser`. In a parser that predates the feature the second option is plain text with stray braces and is on no list. The pipe is one character away from the alias pipe of cooklang-rs (`@milk|almond milk{}`). The parser has to find where a bare name of several words starts.
3. **Group keys for options that sit apart:** `@|milk|milk{200%ml} … @|milk|almond milk{100%ml}`, also in `@tmlmt/cooklang-parser`. Labels already let options sit apart, so this would be a second syntax for the same thing.
4. **A bracketed choice:** `[@water{1L}|@veggie stock{1L}](if you want to add extra flavour)`, from discussion 68. It adds a block and reads as a Markdown link. The reason it carries is kept, as `[reason]` after the option.
5. **Front-matter variables** ([#149](https://github.com/cooklang/spec/pull/149)). The options sit away from the step, so the recipe text no longer names what goes in. An application that predates the feature lists an ingredient named `yeast` for `@yeast{4%g}`, where an option written in the step is still listed under its own name. Variables are for single components by design; that proposal leaves steps and sections to a separate one.
6. **A substitutions file for the whole collection.** Right for personal swaps that hold across recipes. It cannot hold an amount per recipe or a reason.
7. **Another character than `?` for labels.** `?` already means "maybe" in `@?`, and a toggle is the named form of the same idea. The two never meet: the optional marker follows the sigil, and a label follows the component.
8. **Tags in square brackets in front of steps and sections:** `[vegan] Add @oat milk{1%L}`, from discussion 75 and `@tmlmt/cooklang-parser`. Square brackets already hold block comments and, in notes, link text ([#159](https://github.com/cooklang/spec/pull/159)).
9. **`?(vegetarian)` at the start of a step,** from discussion 122. Close to what is proposed. `?vegetarian:` needs no brackets and has the same form on a section and on a component.
10. **Comment directives:** `-- ?sourdough`. Applications that predate the feature hide the label, so the reader sees steps that contradict each other with no hint. Tools that strip comments would delete logic.
11. **Fenced blocks:** `:::sourdough … :::`. Extra lines, and a closing fence that people forget.
12. **A pipe at the start of a line.** Looks like a Markdown table.
13. **A bare `or` line between steps.** It has no name, so the choice cannot reach another section, and it is English.
14. **A keyword:** `If oven:`. It collides with prose ("If the dough is sticky, …") and needs translating.
15. **Choices grouped by position,** where labels that sit next to each other form one choice. Two unrelated toggles in adjacent steps (`?spicy:` then `?garnish:`) become one either-or choice with the first switched on. A real choice interrupted by a note splits into two toggles that are both off, so the bread loses its yeast. Moving a step changes the meaning without a sign. A declared choice also gives the picker its heading, which position cannot.
16. **A reason per group** instead of per option. With two options it is the same thing. With three, each later option may have its own reason.
17. **Per-option `@?`.** Switching from the first option to another would then turn an optional slot into a required one.
18. **Deltas in overrides:** `time: +20 minutes`. That is arithmetic over free-form time strings. A whole value is always right.
19. **A separate file per version.** Right when most of the file differs. For one swapped section the copies drift apart.

## Acknowledgments

Thanks to [@tobei](https://github.com/tobei) for opening the discussion on alternative ingredients; to [@jakeanq](https://github.com/jakeanq), who wrote `@fresh yeast{10g}|@dry yeast{5g}` there in 2022 and opened the discussion on variants; to [@digital-carver](https://github.com/digital-carver) for pointing out that alternatives do not always sit next to each other; to [@FranGarc](https://github.com/FranGarc) for the reason after the alternatives; to [@keithamus](https://github.com/keithamus) for substitutes that replace several ingredients at once; to [@tmlmt](https://github.com/tmlmt), who proposed the reason in square brackets and has implemented alternatives and variants in `@tmlmt/cooklang-parser`; to [@chrbala](https://github.com/chrbala) and [@shicks](https://github.com/shicks) for describing the kinds of variants; to [@philocalyst](https://github.com/philocalyst) for the `?` in front of a step; to [@LukaHummel](https://github.com/LukaHummel) for the component variables proposal; and to the authors of [Gram](https://github.com/gram-lang/gram), whose alternatives use the same pipe.
