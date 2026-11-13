# Optional `%` between quantity and unit

* Proposal: [0024-optional-unit-separator](0024-optional-unit-separator.md)
* Authors: [Alexey Dubovskoy](https://github.com/dubadub)
* Review Manager: TBD
* Status: **Awaiting review**

## Introduction

Today a unit is recognised only when a `%` stands between it and the quantity: `@water{1%cup}`. This proposal makes the `%` optional when the amount is a plain number followed by a unit. `@water{1 cup}` and `@flour{200g}` are read as a number and a unit. The `%` stays valid, stays the form that applications write, and is still needed when the amount is not a plain number and a unit.

Discussion thread: [Consider using space instead of % between amount and unit](https://github.com/cooklang/spec/discussions/54)

## Motivation

**The mistake is silent.** Without the `%`, `@water{1 cup}` is a text quantity with the value `1 cup` and no unit. Every application prints "1 cup", so the author sees nothing wrong. But a text quantity is not a number:

- it does not scale: at twice the servings it is still "1 cup";
- it does not add up: two of them on a shopping list are "1 cup, 1 cup", not "2 cups";
- a timer written `~{10 minutes}` has a text value and no unit, so an application cannot tell how long it runs;
- a reference written `@./Base{4 servings}` is not scaled to four servings.

**People write it this way.** `1 cup` and `200g` are how amounts are written in every recipe a person has read, and the `%` has been questioned in the discussion thread since 2021. A search of public `.cook` files on GitHub finds `{1 cup}`, `{2 tbsp}` and `{200g}` in dozens of files. Recipes converted by language models show the same habit: in a sample of 633 recipes converted from web pages, 119 (19%) had at least one amount written as a number and a unit with no `%`.

**Implementations already disagree.** [cooklang-rs](https://github.com/cooklang/cooklang-rs) reads the space form when its `ADVANCED_UNITS` extension is on. The language server and the desktop editor parse with the extension; [CookCLI](https://github.com/cooklang/cookcli) and the mobile apps parse without it. So the same `{1 cup}` scales in the editor and is fixed text on the phone. The specification should say which one is right.

## Proposed solution

Inside braces, a number followed by a unit is a number and a unit, with or without the `%`.

```cooklang
Mix @flour{200g} with @milk{1 1/2 cups} and @sugar{2 tbsp}.

Rest for ~{10 minutes}.
```

This reads the same as:

```cooklang
Mix @flour{200%g} with @milk{1 1/2%cups} and @sugar{2%tbsp}.

Rest for ~{10%minutes}.
```

| Written | Today | Proposed |
|---|---|---|
| `{1 cup}`, `{200 g}`, `{1.5 l}` | text | 1 `cup`, 200 `g`, 1.5 `l` |
| `{1/2 cup}` | text | 1/2 `cup` |
| `{200g}`, `{2tbsp}`, `{3.5L}` | text | 200 `g`, 2 `tbsp`, 3.5 `L` |
| `{2 large}`, `{1 medium can}`, `{2 fl oz}` | text | 2 `large`, 1 `medium can`, 2 `fl oz` |
| `~{10 minutes}`, `~{10min}` | text, no unit | 10 `minutes`, 10 `min` |
| `{1 cup + 2 tablespoons}`, `{8 oz/250 g}` | text | text |
| `{3/4 cup, lightly crushed}` | text | text |
| `{a pinch}`, `{to taste}`, `{few sprigs}` | text | text |
| `{1%cup}`, `{few%sprigs}`, `{2%14-oz cans}` | as written | unchanged |

Anything that contains a `%` is read exactly as it is today.

## Detailed design

### The rule

Read the content of the braces, after the optional `=` that [locks the quantity](../conventions.md):

1. **If it contains a `%`, nothing changes.** The first `%` separates the quantity from the unit.
2. **Otherwise,** take the longest prefix that is a number. If what follows the number, after optional whitespace, starts with a letter or `°` and contains no digit, no comma and no bracket, then the prefix is the quantity and the rest, trimmed, is the unit.
3. **Otherwise** the content is read as today: a bare number, or a text quantity.

A *number* is whatever is already read as a numeric quantity when it stands before a `%`: an integer, a decimal, or a fraction such as `1/2` or `1 / 2`. This proposal does not change what a number is. If [ranges](https://github.com/cooklang/spec/pull/109), mixed numbers or other forms are added to the language, they take this rule with no further change, so `{1 1/2 cups}` works wherever `{1 1/2%cups}` does.

A *letter* is any Unicode letter, so `{1 стакан}` and `{3个}` are amounts. `°` is allowed so that `{180 °C}` is one.

The *unit* is the rest of the brace, as it is after a `%`. It may be several words: `{2 fl oz}` and `{2%fl oz}` are the same amount.

Whitespace between the number and the unit is optional: `{200g}` and `{200 g}` are the same amount.

### The guard

The rule in step 2 is deliberately narrow. When the brace holds more than a number and a unit, it stays text, as it is today, and the author can add a `%` to say what was meant.

- **A digit in the rest** means a second number: `{1 cup + 2 tablespoons}`, `{8 oz/250 g}`, `{2 x 400 g}`, `{1 to 2 tsp}`, `{13 by 9-inch}`. Read as the number 1 with the unit `cup + 2 tablespoons`, the first of these would scale to "2 cup + 2 tablespoons". As text it keeps saying what the author wrote.
- **A comma or a bracket in the rest** means the brace holds a comment as well: `{3/4 cup, lightly crushed}`, `{1 can (drained)}`.
- **A prefix that is not a number** is not an amount: `{1 000 g}`, `{2 14-oz cans}`, `{12-inch}`, `{9x13}`, `{01 cup}`.

### Where the rule applies

Everywhere a quantity and a unit are written:

- **Ingredients and timers** in recipes.
- **Cookware.** A unit after a cookware quantity is read the same way with or without the `%`: `#tin{30cm}` and `#tin{30%cm}`.
- **Recipe references.** `@./Base{4 servings}` and `@./sauces/Hollandaise{150 ml}` are scaled as the [`%` forms](../conventions.md#scaling-referenced-recipes) are.
- **`.menu` files,** which are Cooklang files.
- **`.shopping-list` files:** `olive oil{4 l}` and `? chilli flakes{2 pinch}`.
- **The `yield` metadata key:** `yield: 500 ml` and `yield: [500 ml, 350 g]`.
- **Pantry configuration:** `rice = "5 kg"`, `quantity = "500 ml"`, `low = "200g"`.

One rule in all of these matters more than the rule itself. Someone who learns `{500 g}` in a recipe will write `yield: 500 ml` next.

### Grammar

`number` is new in the grammar: today `quantity` is free text and the grammar does not say which quantities are numbers.

```ebnf
amount     = ( number, [ white space ], bare units )
           | ( quantity, "%", units )
           | quantity ;

bare units = ( letter | "°" ), { text item - digit - "," - "(" - ")" - "%" - "}" } ;

number     = integer | decimal | fraction ;
fraction   = integer, [ white space ], "/", [ white space ], integer ;
integer    = digit - "0", { digit } | "0" ;
decimal    = integer, ".", { digit }- ;

letter     = ? Unicode General Category L* ? ;
digit      = ? Unicode General Category Nd ? ;
```

The first alternative is tried first and applies only when the whole content of the braces matches it.

### What applications write

Applications keep writing the `%` form when they create or change a file. This includes converters, formatters, shopping list and pantry writers, and editor completion. A file written by a tool should be read correctly by an application that has not been updated, and the `%` form is the one every implementation reads.

An application must not rewrite an author's `{1 cup}` to `{1%cup}` unless it was asked to.

### Backward compatibility

Nothing that contains a `%` changes. Nothing without a number changes. The amounts that change are the ones written as a number and a unit with no `%`. They are text quantities today and become a number and a unit.

For those, what the reader sees stays the same, except that a glued amount is printed with a space (`500g` as "500 g"). What changes is that the amount scales, converts and adds up. Two readings are odd but harmless: `{10 or so}` becomes 10 with the unit `or so`, and `{2x}` becomes 2 with the unit `x`.

One existing canonical test changes. `testQuantityDigitalString` expects `@water{7 k }` to be the quantity `7 k` with no units; under this proposal it is the quantity 7 with the unit `k`.

**Old applications reading new files.** An application that predates this proposal reads `{1 cup}` as text: it is shown correctly and is not scaled or added up. That is what the same file does today, so nothing gets worse, and it is the reason applications keep writing the `%` form.

**cooklang-rs with `ADVANCED_UNITS`.** Applications that parse with the extension already read the space form. For them two things change: the guard (`{1 cup + 2 tablespoons}` is text again, where the extension reads the unit `cup + 2 tablespoons`), and the glued form `{200g}`, which the extension does not read. The extension also checks timer units and the units of repeated ingredients; those checks are not part of this proposal.

### Canonical cases

| Source | Quantity | Units |
|---|---|---|
| `@water{1 cup}` | 1 | `cup` |
| `@flour{200g}` | 200 | `g` |
| `@milk{1/2 cup}` | 0.5 | `cup` |
| `@milk{1 / 2 cup}` | 0.5 | `cup` |
| `@oil{1.5 l}` | 1.5 | `l` |
| `@tomatoes{1 medium can}` | 1 | `medium can` |
| `@мука{2 ст. ложки}` | 2 | `ст. ложки` |
| `@water{7 k }` | 7 | `k` |
| `@yeast{=7 g}` | 7, locked | `g` |
| `~{10 minutes}` | 10 | `minutes` |
| `~rest{5min}` | 5 | `min` |
| `@butter{1 cup + 2 tablespoons}` | `1 cup + 2 tablespoons` | none |
| `@onions{3/4 cup, lightly crushed}` | `3/4 cup, lightly crushed` | none |
| `@rice{1 000 g}` | `1 000 g` | none |
| `@beans{2 14-oz cans}` | `2 14-oz cans` | none |
| `@tin{9x13}` | `9x13` | none |
| `@milk{01 cup}` | `01 cup` | none |
| `@thyme{few sprigs}` | `few sprigs` | none |
| `@thyme{few%sprigs}` | `few` | `sprigs` |
| `@beans{2%14-oz cans}` | 2 | `14-oz cans` |
| `@cream{80%}` | 80 | none |

## Effect on applications which use Cooklang

Most of the change is in the parser. Applications that take their quantities from the parser need a parser update and nothing else. Code that splits an amount on `%` by itself, as readers of `yield`, menus, shopping lists and pantry files often do, needs to call one shared function instead.

### CookCLI (terminal and web-server)

1. Recipes, menus and shopping lists read the new form: scaling, unit conversion, shopping list totals and pantry matching work for `{1 cup}` as they do for `{1%cup}`.
2. `yield` is read with or without the `%`, so `@./sauces/Hollandaise{150 ml}` against `yield: 500 ml` scales.
3. `cook fmt`, the shopping list and the pantry commands keep writing the `%` form.
4. `cook doctor` can note an amount that looks like a number and a unit but stayed text because of the guard, with the hint to add a `%`.

### Mobile applications

1. A parser update. No change to any screen.
2. Recipes that were written or imported with `{1 cup}` start to scale and to add up on the shopping list.
3. Timers written `~{10 minutes}` get a duration and a unit.

### Editors, the language server and syntax highlighting

1. Highlight the unit when there is no `%`. Grammars that look for a literal `%` (TextMate, Sublime, Prism and others) need a second pattern.
2. Completion keeps inserting `quantity%unit`.
3. Applications that parse with `ADVANCED_UNITS` pick up the guard and the glued form.

### Other parsers

Parsers that are not built on cooklang-rs follow the canonical tests. Until they do, they read the new form as text, as they do today.

## Open questions

1. **Cookware in the canonical tests.** The canonical result for cookware has a `quantity` and no `units`. The tests need a decision on what `#tin{30cm}` and `#tin{30%cm}` yield there.
2. **A unit that is not a unit.** `{10 or so}` and `{3 per serving}` pass the guard. A list of known units would catch them, but the language has never had one, and `{2%or so}` reads the same way today.
3. **Whether a formatter normalises.** `cook fmt` could rewrite `{1 cup}` to `{1%cup}`, leave it, or take an option.
4. **Order with ranges and other number forms.** Independent of this proposal, but `{1 to 2 tsp}` and `{1½ cups}` only work once those numbers are defined.

## Alternatives considered

1. **Keep requiring the `%`.** Converters can insert it, and some do. That covers files that pass through a converter. A recipe typed by hand does not, and the failure stays silent.
2. **Adopt the cooklang-rs extension as it is.** It requires whitespace, so `{200g}` stays text, and it has no guard, so `{1 cup + 2 tablespoons}` becomes the number 1 with a unit that scales wrongly.
3. **Another separator.** `*` was tried in the discussion thread in 2021 and dropped: next to `%` it looks the same once highlighted, and it would break every existing recipe. It also does nothing for the author who writes no separator at all.
4. **Require whitespace between number and unit.** Simpler to describe, but `200g` is a common way to write a metric amount, and it would then behave differently from `200 g` for no reason an author could see.
5. **Limit the unit to one word.** Covers most cases, but not `fl oz` or `ст. ложки`, and it would make `{2 fl oz}` differ from `{2%fl oz}`.
6. **Remove the `%`.** It is in most quantities ever written and in the output of every tool. It is also the only way to give a unit to a quantity that is a word (`{few%sprigs}`) or to write a unit with a digit in it (`{2%14-oz cans}`).

## Acknowledgments

Thanks to [@DusteDdk](https://github.com/DusteDdk) for opening the discussion; to [@vborovikov](https://github.com/vborovikov), whose "the unit starts at the first letter" is the rule proposed here; to [@chrbala](https://github.com/chrbala) for raising mixed numbers; to [@aspiers](https://github.com/aspiers) for arguing that an amount should be readable as it is written; and to [@Zheoni](https://github.com/Zheoni), who implemented the space form in cooklang-rs.
