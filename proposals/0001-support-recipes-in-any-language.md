# Support recipes in any language

* Proposal: [0001-support-recipes-in-any-language](0001-support-recipes-in-any-language.md)
* Authors: [Alexey Dubovskoy](https://github.com/dubadub)
* Review Manager: TBD
* Status: **Awaiting review**

## Introduction

In many languages a noun changes its form inside a sentence. Cooklang lists every ingredient, cookware item and unit exactly as it was written in the step, so in those languages the ingredient list shows the wrong form, the same ingredient appears on the shopping list more than once, and scaled amounts are printed with the wrong plural.

This proposal gives authors three places to say what the dictionary form of a word is: next to the word (`@муку:мука{600%г}`), in the recipe's front matter (`forms`, `plurals`), and in a table for the whole collection (`config/forms.conf`). It also says how a tool picks the form of a word that goes with a number. No tool has to know the grammar of any language: every form comes from the author or from a table that people keep.

Discussion threads: [Support languages other than English](https://github.com/cooklang/spec/discussions/144), [Alternative writing style for ingredients for listing](https://github.com/cooklang/spec/discussions/85), [separate text string from list string](https://github.com/cooklang/spec/discussions/123)

## Motivation

A Russian recipe:

```cooklang
Просеять @муку{600%г} в #миску{}.
Добавить @чеснок{2%зубчика}.
```

("Sift the flour into a bowl. Add 2 cloves of garlic.") The words are in the accusative and genitive, as the sentence needs. Today's tools produce:

```
Ingredients:
  муку     600 г          should be: мука
  чеснок   2 зубчика
Cookware:
  миску                   should be: миска
```

Listing `муку` is like listing "of flour" in English. And it gets worse:

1. **Duplicates.** A second recipe that writes `@мука{200%г}` or `@муки{100%г}` adds a second and a third line to the shopping list, because the three forms are three names.
2. **Units do not add up.** `2%зубчика` and `5%зубчиков` are two different units to a parser, so the list says `2 зубчика, 5 зубчиков` and not 7.
3. **Scaling prints the wrong form.** The recipe above scaled by three says `6 зубчика`. Russian needs `6 зубчиков`.

The same has been reported for Finnish (`maidon` in the sentence, `maito` in the list), Polish (`miętę` / `mięta`), Hungarian (`almával` / `alma`, where the stem itself changes, so the trick of writing `@alma{}val` does not work), German and the Slavic languages. English has a small version of it: `@tomatoes{2}` and `@tomato{1}` are two lines on a shopping list.

There are two workarounds today. One is to write every sentence so that the ingredient stays in its dictionary form, which makes the text unnatural. The other is to list the forms as synonyms in `aisle.conf`, which merges them on the shopping list in some tools, but does nothing for the recipe's own ingredient list, for cookware or for units.

## Proposed solution

There are two jobs:

* **Reading.** Map the form written in the sentence back to the dictionary form. This fixes the lists and lets amounts add up.
* **Printing.** When a tool prints a number it has computed, by scaling or by adding, pick the form of the word that goes with that number.

The dictionary form can be given at three levels.

### 1. Inline, with a colon

```cooklang
Просеять @муку:мука{600%г} в #миску:миска{}.
Sekoita ainekset @maidon:maito{2%dl} kanssa.
Keverd össze @almával:alma{2}.
```

Left of the colon is the word as the step shows it. Right of the colon is the name that goes on the ingredient list, the cookware list and the shopping list.

The same form covers a short name in the sentence for a long name on the list, in any language:

```cooklang
Stir in the @soup:condensed cream of chicken soup{1%can}.
```

### 2. In the recipe's front matter

```cooklang
---
locale: ru
forms:
  мука: [муку, муки]
  миска: [миску, миске]
plurals:
  зубчик: {one: зубчик, few: зубчика, many: зубчиков, other: зубчика}
---

Просеять @муку{600%г} в #миску{}.
Добавить @чеснок{2%зубчика}.
```

`forms` is for reading: the key is the dictionary form, the list holds the forms that may be written. `plurals` is for printing: it says which form goes with which number. The step text stays free of extra markup, and a parser that does not know these keys shows the recipe exactly as it does today.

### 3. For the whole collection

`config/forms.conf`, next to `aisle.conf`:

```
[ru]
мука|муку|муки
миска|миску|миске
зубчик|one=зубчик|few=зубчика|many=зубчиков|other=зубчика

[fr]
tomate|one=tomate|other=tomates
oignon|oignons
```

Someone who writes all their recipes in one language keeps one table and writes plain `@муку{600%г}` everywhere. A table can be shared between people who cook in the same language.

With any of the three in place, the recipe above gives:

```
Ingredients:
  мука     600 г
  чеснок   2 зубчика
Cookware:
  миска
```

and scaled by three, `чеснок 6 зубчиков`.

## Detailed design

### Grammar

The name of an ingredient, cookware item or timer written with braces may hold a written form and a name, separated by a colon:

```ebnf
multiword component  = [ written form, ":" ], name, "{", [ amount ], "}" ;
written form         = { text item - "{" }- ;
name                 = { text item - "{" }- ;
```

The productions are a sketch: which colon separates the two sides is decided by rule 2 below.

Rules:

1. The form needs braces. Without braces a name ends at the first punctuation character, as today: `@tomates:tomate` is the ingredient `tomates` followed by the text `:tomate`.
2. The separator is the first colon in the name that is not followed by white space and is not between two digits. Any other colon is part of the name. `@fresh herbs: mint, basil{}`, `@gluten free 1:1 baking flour{}` and `@80:10:10{}` keep their meaning.
3. Both sides may have several words. If either side is empty, the colon is part of the name and the tool should report it.
4. Markers stay in front of the written form: `@?сметаной:сметана{}`.
5. Timers accept the form so that the three sigils share one grammar. The step shows the left side. This proposal gives the right side of a timer no further meaning.

### Front matter

Two new keys in the canonical metadata table.

| Key | Purpose | Value |
|---|---|---|
| `forms` | Reading | A mapping. Each key is a dictionary form; each value is a written form or a list of written forms |
| `plurals` | Printing | A mapping. Each key is a dictionary form; each value maps plural categories to forms. The categories are the six of the [Unicode CLDR plural rules](https://cldr.unicode.org/index/cldr-spec/plural-rules): `zero`, `one`, `two`, `few`, `many`, `other` |

Both keys apply to ingredient names, cookware names and units, in one name space. A form listed under `plurals` is also read as a written form of its key, so it does not have to be repeated under `forms`.

### `config/forms.conf`

The file uses the line syntax of `aisle.conf`. Each section is named after a language, in the format of `locale`. On each line the first entry is the dictionary form and the other entries are written forms. An entry written `category=form` is also the count form for that plural category, the same as an entry under `plurals`. A line that starts with `--` is a comment.

A [sample file](../examples/config/forms.conf) has a few common ingredients, counting units and cookware for thirteen languages. It is a starting point for review by speakers of those languages, not a finished table.

The file is looked up the way `aisle.conf` is: in the collection's `config` directory first, then in the user's global configuration directory.

Which sections apply to a recipe:

* A recipe with `locale: pt_BR` uses the section `[pt_BR]`, then `[pt]`.
* A recipe with no `locale` uses every section. If two sections map a written form to different dictionary forms, the tool reports it and leaves the word as written.

`forms.conf` and the synonyms in `aisle.conf` are different tables. A line in `forms.conf` says "this is the same word in another form" and applies wherever a name is listed. A synonym line in `aisle.conf` (`yellow onion|onion`) says "buy these as one thing" and applies to the shopping list. Aisle and pantry lookups use the dictionary form.

### Reading: resolving a name

For each ingredient name, cookware name and unit, a tool takes the first that applies:

1. The name after an inline colon.
2. The dictionary form from the recipe's `forms` or `plurals`.
3. The dictionary form from `forms.conf`.
4. The text as written, as today.

Details:

* The whole name is compared, ignoring letter case. Names are not matched word by word: `лавровым листом` needs its own entry under `лавровый лист`.
* The lookup is done once. Its result is not looked up again.
* A written form that one layer lists under two dictionary forms is reported and not mapped.
* Units are resolved before any unit table is consulted, so `2%зубчика` and `5%зубчиков` become one unit and add up.

Parsers expose both values for every component: the name (the dictionary form, spelled as it was declared) and the written form. The step text always shows the written form. Ingredient lists, cookware lists, shopping lists, and every lookup by name use the name.

In the canonical test format:

```yaml
testIngredientWrittenForm:
  source: |
    Просеять @муку:мука{600%г}
  result:
    steps:
      -
        - type: text
          value: "Просеять "
        - type: ingredient
          name: "мука"
          written: "муку"
          quantity: 600
          units: "г"
    metadata: {}
```

`written` may be omitted when it equals `name`, so existing tests do not change.

### Printing: the form that goes with a number

Where a tool prints a number directly before a word (an amount and its unit, or a count and a name when there is no unit), it:

1. Finds the word's count forms in the recipe's `plurals`, then in `forms.conf`.
2. Finds the plural category of the number with the CLDR plural rules of the language. The language is the recipe's `locale`, or else the section of `forms.conf` in which the word was found.
3. Prints the form declared for that category. If none is declared it prints the `other` form, and if that is missing too, the dictionary form.
4. For a range, uses the upper end.
5. For a fraction below one that is printed as a fraction (`1/2`, `½`), uses the `one` form in English, German, Dutch, Spanish and Italian: `1/2 cup`, `½ Prise`. The CLDR rules put 0.5 in `other` for these languages, which is right for a decimal (`0.5 cups`) and wrong for a fraction. Other languages follow the CLDR category (`1/2 стакана`, `1/2 sipulia`).

Count forms for units come from these tables like those of any other word. A unit table may know that `cup` and `cups` are one unit; it does not say which of them goes with a number.

A word with no count forms, or with no known language, is printed as the author wrote it, as today.

| Recipe | Printed |
|---|---|
| `@чеснок{2%зубчика}` | `2 зубчика` |
| the same, scaled by 3 | `6 зубчиков` |
| `2%зубчика` and `5%зубчиков` added | `7 зубчиков` |
| the same, scaled to 21 | `21 зубчик` |
| the same, scaled to 1.5 | `1.5 зубчика` (fractions are `other` in Russian) |
| `@tomates:tomate{3}`, `locale: fr`, `tomate: {one: tomate, other: tomates}` | `3 tomates` |
| the same, scaled to 1.5 | `1.5 tomate` (French counts 1.5 as `one`) |
| `@onions{3}`, `locale: en`, `onion: {one: onion, other: onions}`, scaled to 1 | `1 onion` |
| `@flour{1%cup}`, `cup: {one: cup, other: cups}`, scaled by 3 and by 0.5 | `3 cups`, `1/2 cup` |

Tools must not derive a form by rule. Every printed form was written by the author or by the keeper of the table. The only language data a tool needs is the CLDR plural rule set, which is available as a library on every platform.

The text of a step is not rewritten when a recipe is scaled.

### `locale`

`locale` keeps its definition: an ISO 639 language code, optionally followed by an underscore and an ISO 3166 country code. A hyphen in place of the underscore (`pt-BR`) is read the same way. Tools must not guess the language of a recipe from its text or from the language of the device.

### Reports

Tools should report, without failing:

* a written form listed under two dictionary forms;
* an empty side of an inline colon;
* a key under `plurals`, or a `category=` prefix, that is not one of the six categories.

### Backward compatibility

* **Names with a colon.** Under the current specification `@муку:мука{}` is an ingredient named `муку:мука`. After this change it is the ingredient `мука`. A name with a colon that is followed by a space, or that stands between digits, does not change. In a set of 16,268 machine-written English recipes, eight names contained a colon and all eight are of those two kinds.
* **Front matter.** `forms` and `plurals` become reserved keys.
* **Older parsers.** A recipe that uses only front matter or `forms.conf` is read by an older parser as today, as long as it accepts nested values in front matter (checked with CookCLI 0.38.1). A recipe that uses the inline form shows `муку:мука` in the step and on the list of an older parser; nothing is lost, but it reads badly. Recipes meant for wide sharing should prefer the front matter until parsers have caught up.
* **The cooklang-rs alias extension.** cooklang-rs has an extension in which `@white wine|wine{}` means name first, display text second, the reverse of the order here. It is not part of the specification. [Proposal 0025](https://github.com/cooklang/spec/pull/162) uses the pipe to separate alternatives, so the extension should be retired in favour of the colon: `@name|display{}` becomes `@display:name{}`.

## Effect on applications which use Cooklang

### Specification updates

Once accepted: the colon form in the language specification and the EBNF, with canonical tests; `forms` and `plurals` in the metadata table; a "Word forms" section in the conventions for `forms.conf`, the resolution order and the printing rule.

### Display

1. Steps show the written form.
2. Ingredient and cookware lists show the dictionary form, and add up amounts of the same dictionary form.
3. Amounts that a tool computed use the count form when one is declared.

### CookCLI (terminal and web-server)

CookCLI should:

1. Look for `forms.conf` where it looks for `aisle.conf`.
2. Use dictionary forms in every recipe output and in `shopping-list`; expose the written form in JSON and YAML output.
3. Use count forms for scaled and added amounts.
4. Show the reports above in `cook doctor`.

No new sub-command is required.

### Mobile applications

Mobile apps should:

1. Show dictionary forms in the ingredient list, the cookware list and the shopping list.
2. Use count forms when the user changes the number of servings.
3. Read `config/forms.conf` from the synced collection.

### Editors

Editors and the language server should highlight both sides of the colon, offer known dictionary forms after it, show the reports as diagnostics, and may offer to add a word to `forms`.

### Parsers

Parsers need the grammar change, the `written` property, the two metadata keys, a loader for `forms.conf`, and one resolve step after parsing that takes the tables as input. Printing needs a CLDR plural rules library. A parser that implements reading without printing is still useful: reading is what fixes the lists.

## Open questions

1. **The colon rule.** Is "not followed by white space, not between digits" the right test for the separator, or should the separator be another character?
2. **Recipes with no `locale`.** The draft applies every section of `forms.conf`. The other choice is a default language for the collection, which needs a setting that does not exist yet.
3. **Count forms in `forms.conf`.** Is `category=form` on the same line readable enough for languages with three or four forms?
4. **Shared tables.** Should there be a common place for ready-made `forms.conf` sections per language?
5. **Fractions.** Which languages use the singular after a fraction below one? The draft lists English, German, Dutch, Spanish and Italian.
6. **English.** The draft has no "add an s" rule, because it is wrong for tomato, leaf and loaf; English count forms are declared like any others, and the sample file has an `[en]` section. Should tools be allowed to pluralise English by rule when nothing is declared?
7. **Words this cannot express.** Speakers of the languages mentioned above are asked to try their recipes against the rules and report what does not fit.

## Alternatives considered

### A morphology engine in the parser

A parser could carry the grammar of each language and find the dictionary form by itself. That is a large amount of data per language, it is wrong on rare words and brand names, two parsers with different engines would disagree, and a wrong guess silently renames an ingredient.

### The form in parentheses: `@mięta{}(miętę)`

Proposed in [discussion #123](https://github.com/cooklang/spec/discussions/123). Parentheses after the braces already hold the preparation note.

### A pipe: `@maidon|maito{2%dl}`

Proposed in [discussion #85](https://github.com/cooklang/spec/discussions/85). The pipe separates alternatives in proposal 0025 and alternative units in proposal 0021.

### Only a table for the collection

Suggested in the comments on this proposal. It keeps the recipe text clean, but a recipe sent to someone else loses its forms. It is the third level here.

### Only the inline form

Every mention carries both words, and an older parser shows both. It is the first level here.

### One front-matter key for reading and printing

A single `forms` key whose values are either a list or a category mapping. Two keys were chosen so that each has one job.

### Reuse the synonyms of `aisle.conf`

It already merges forms on the shopping list in CookCLI. But a synonym line such as `yellow onion|onion` would then rename onions inside every recipe, and cookware and units have no aisle.

### Locale in the file name

An earlier draft allowed `Koldskål.da.cook`. The file name is the recipe's title in every tool, and `locale` in the front matter already says the same.

### Out of scope

Number formats, translated unit names, translated application text, agreement of adjectives, and translation of recipes are not part of this proposal.

## Acknowledgments

[MiK546](https://github.com/MiK546) described the problem and the inline form in 2022. [michalmoc](https://github.com/michalmoc), [FlyinPancake](https://github.com/FlyinPancake) and [egregors](https://github.com/egregors) gave the Polish, Hungarian and Russian cases. [rentib](https://github.com/rentib) suggested a configuration file in the style of `aisle.conf`, which became the third level. The colon as the separator follows [Gram](https://github.com/gram-lang/gram).
