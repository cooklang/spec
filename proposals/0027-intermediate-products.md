# Intermediate products

* Proposal: [0027-intermediate-products](0027-intermediate-products.md)
* Authors: [Alexey Dubovskoy](https://github.com/dubadub)
* Review Manager: TBD
* Status: **Awaiting review**

## Introduction

A recipe is a graph: the pastry and the filling are made apart and meet in the tin, and the stock is made once and used twice. Cooklang writes a recipe as a list of steps, and nothing in the file says which step feeds which. This proposal adds two pieces of syntax. `-> name` at the end of a step, or on a section line, names what that step or section makes. `@&name{}` in a later step uses it. A named thing of this kind is called a **product** below: the dough, the stock, the egg whites. A product is never on the shopping list, and from the two marks an application can build the graph of the recipe without reading the prose.

The proposal has four parts. Part A, naming a product and using it, can be accepted and implemented without the others. Parts B (part of a product), C (two products from one step, and the amount made) and D (what nobody uses) each need Part A and nothing else.

Discussion threads: [syntax for tracking combining of ingredients](https://github.com/cooklang/spec/discussions/62), [Tree / timeline representation of the cooking process](https://github.com/cooklang/spec/discussions/48), [Ingredient References Across Sections](https://github.com/cooklang/spec/discussions/125), [Break down recipe instructions further](https://github.com/cooklang/spec/discussions/59), and [Diagram generation](https://github.com/cooklang/cookcli/issues/410) in CookCLI.

## Motivation

**An application cannot draw the recipe.** A flow diagram, a table in the style of Cooking for Engineers, a timeline, and a plan for two cooks all need the same fact: which steps feed which. Today that fact is only in the prose.

**The author has two bad ways to mention the dough.**

```cooklang
Mix @flour{500%g} and @water{350%g} in a #bowl{}.

Knead the dough for ~{10%minutes}.
```

Left unmarked, "the dough" is a word like any other. Marked as an ingredient, `@dough{}`, it goes on the ingredient list and the shopping list, and nobody can buy it.

**Amounts of a product cannot be written.** "Add 100 g of the levain", "half the mash", "a ladle of stock at a time" and "keep 250 ml of the pasta water" are each an amount of something the recipe made. There is no token to hold them.

**The one existing form counts steps.** cooklang-rs has an extension, [intermediate preparations](https://github.com/cooklang/cooklang-rs/blob/main/extensions.md#intermediate-preparations), which refers to a step or a section by its position: `@&(~1)dough{}` is "the dough from one step back", `@&(2)dough{}` is "from step 2". It keeps the product off the lists, and the [tabular-recipes](https://github.com/johnhkchen/tabular-recipes) project draws tables with it. A position has three costs. A step put in between changes what every later count means. The reference cannot leave its section. And a reader of the file has to count steps to know what is meant.

## Proposed solution

Two pieces of syntax:

- **`-> name`** at the end of a step, or last on a section line, names what that step or section makes.
- **`@&name{}`** in a later step uses it. `@&name{1/2}` and `@&name{100%g}` use part of it.

| Part | What it adds | Form |
|---|---|---|
| A | A step's product, a section's product, and the use of one | `… -> batter`, `= Dough -> dough`, `@&dough{}` |
| B | Part of a product | `@&levain{100%g}`, `@&mash{1/2}` |
| C | Two products from one step; the amount made | `-> yolks + whites`, `-> stock{2%l}` |
| D | What nobody uses | no syntax, one rule |

All four in one file:

```cooklang
---
title: Lemon Meringue Tart
servings: 8
---

Separate @eggs{4}. -> yolks{4} + whites{4}

= Shell -> shell

Rub @butter{120%g}(cold, cubed) into @plain flour{220%g} and @icing sugar{40%g}. Bind with @&yolks{1} and @cold water{2%tbsp}.

Chill for ~{30%minutes}, then roll out and line a #tart tin{}.

Bake blind at 180 °C for ~{25%minutes}.

= Curd -> curd{600%g}

Whisk @lemon juice{150%ml} with @caster sugar{150%g}, the @&yolks{} and @eggs{2} in a #saucepan{}.

Cook over a low heat for ~{8%minutes} until thick, then beat in @butter{80%g}.

= Assemble

Fill the @&shell{} with the @&curd{} and chill for ~{2%hours}. -> tart

Whisk @&whites{3} with @caster sugar{150%g} to stiff peaks. -> meringue

Spread the @&meringue{} over the @&tart{} and brown under a #grill{}.
```

What an application reads from it:

| Step | Uses | Makes |
|---|---|---|
| Separate | eggs | yolks (4), whites (4) |
| Shell 1, rub and bind | butter, flour, icing sugar, water, 1 of the yolks | |
| Shell 2, chill and line | Shell 1 | |
| Shell 3, bake | Shell 2 | shell |
| Curd 1, whisk | lemon juice, sugar, eggs, the 3 yolks that are left | |
| Curd 2, cook | Curd 1, butter | curd, 600 g |
| Assemble 1, fill | shell, curd | tart |
| Assemble 2, whisk | sugar, 3 of the whites | meringue |
| Assemble 3, spread | meringue, tart | the dish |

The shopping list holds 6 eggs, 200 g of butter, 300 g of caster sugar and the other raw ingredients. It holds no yolks, whites, shell, curd, tart or meringue. One egg white is left over, and the application says so.

## Detailed design

### Part A: naming a product and using it

1. **A step's product.** `-> name` as the last thing in a step, with white space before the arrow, names what the step makes. Everything from that arrow to the end of the step is the tail. The tail is not part of the step's text. A step may run over several lines, so the tail may sit alone on the step's last line. When a step holds several arrows, the last one starts the tail and the others are text. An arrow with no white space before it is text.
2. **A section's product.** `= Dough -> dough` and `== Dough -> dough ==` name what the section's last step makes. The section's name is "Dough". The arrow and the name come last on the line.
3. **Names.** A name is any text without `{`, `}` or a line break, with the white space at its ends removed. It may be several words. It may not begin with `@`, `#`, `~`, `&` or `?`. Names are compared without regard to case.
4. **Using a product.** `@&name{}` in a step uses the product. A one-word name needs no braces (`@&dough`), as for an ingredient. With the optional marker of [0018](0018-optional-components.md) the order is `@?&name{}`. The step shows the name. The product is not on the recipe's ingredient list, not on the shopping list, and not matched against a pantry.
5. **One name, one thing.** In one file a name belongs to one product, and a product may not have the name of an ingredient or of a cookware item in that file. A clash is reported. A name may be given twice in three cases:
   - by sections or steps that are alternatives of each other under [#162](https://github.com/cooklang/spec/pull/162), if that proposal is accepted: `= Pastry ?homemade -> pastry` and `= Pastry ?shop-bought -> pastry`;
   - by a step that uses the whole product and hands the name to what it makes: `Drain the @&pasta{}. -> pasta + pasta water{250%ml}`;
   - by a step inside a section that repeats the section's own name. It changes nothing.
6. **Made before it is used.** A product is named earlier in the file than its first use. A use with no product named before it is reported. Names hold for the whole file: a step's product can be used from another section.
7. **Inside its own section.** In `= Dough -> dough`, a step may say `@&dough{}`. It means what the section has made so far, which is the result of the step before it. In the section's first step nothing is made yet, and the use is reported.
8. **A name goes on.** When a step uses the whole of a product and names no product of its own, the name goes on to what that step makes: after `Knead the @&dough{}.`, the dough is the kneaded dough. When two products are used whole in such a step, both names go on to the one result. When the step does name a product (`Mix the @&wet mix{} with the eggs. -> batter`), the old name is used up. A later use of a used-up name is reported, and is linked to the step that used it up.
9. **Steps without a mark.** A step carries on from the step before it in the same section, unless that step named a product. A named product is set aside until a step uses it by name, so the step after it starts something new. The first step of a section starts something new. A file with no marks is therefore one line of steps per section, which is how it reads today.

Rule 9 has a cost and a limit. The cost: to start something new in the middle of a section, the author names what the step before it made. The tart above names the filled tart for that reason, so that whisking the whites does not carry on from it. The limit: a step that gets equipment ready ("preheat the oven", "butter the tin") is drawn in the line with the steps around it, because nothing in the syntax tells it from "bake for 20 minutes". An author can put such steps in a section of their own. See [Open questions](#open-questions).

### Part B: part of a product

10. **An amount with a unit** takes that much: `@&levain{100%g}`, `@&stock{500%ml}`. The rest stays under the same name.
11. **A number with no unit** is a share of the whole product: `@&mash{1/2}`, `@&dough{0.25}`. One exception: when the product was named with a count (`-> rolls{12}`, rule 15), a number with no unit counts out of it, so `@&rolls{6}` is six rolls and `@&yolks{1}` is one yolk.
12. **Empty braces take what is left.** After `@&whites{3}` out of four, `@&whites{}` is the fourth. When nothing was taken before, it is the whole product.
13. **Scaling.** A share does not change when the recipe is scaled. An amount with a unit, a count, and the amount written with a product's name (rule 15) scale as an ingredient's amount does.
14. **Checks.** When the amount made is known (rule 15) and the units can be compared, uses that add up to more than was made are reported. When it is not known, the amounts are kept as written and nothing is checked. A share above 1 is reported.

### Part C: two products, and the amount made

15. **The amount made.** A name in a tail may carry an amount in braces, written as an ingredient's amount is: `-> stock{2%l}`, `= Curd -> curd{600%g}`, `-> rolls{12}`. It is what the author measured. An application that works out weights uses it in place of its own figure.
16. **Two or more products.** `-> yolks + whites` names two products of one step. The `+` has white space on both sides. Each name may carry its own amount: `-> pasta + pasta water{250%ml}`. A section line names one product only.
17. **How the amounts fall.** An application takes the amounts that are written. This proposal does not say how a step's result is divided among products with no amount.
18. **A section's last step.** When a section line names a product and the section's last step names one too, both names mean the same thing. When that step names two or more, it cannot be told which one the section means, and it is reported.

### Part D: what nobody uses

19. **The dish** is what the last step of the file makes. Its amount is `servings` and `yield` in front matter, as today.
20. **A by-product** is a named product, or what is left of one, that no later step uses: the egg white in the tart above, a curry paste that keeps a week. An application lists by-products with the recipe ("also makes: 1 egg white"). Their ingredients stay on the shopping list. An application that works out nutrition does not count them in the dish. There is no mark for a by-product. A mistyped name shows up in the same list.
21. **Nothing else is reported.** A step whose result has no name and is never used is how every file without marks reads today. It is not reported.

### The graph

An application builds the graph from four things:

- a node for each step;
- an edge from the step that made a product to each step that uses it, with the amount when one is written;
- an edge from a step to the next step in its section, when the first named no product (rule 9);
- raw ingredients, cookware and timers on the step that names them, as today.

A product is always named before it is used (rule 6), so the graph has no cycles. An application may draw it as a tree, a table or a chart. When a product has two users, a table cannot hold it, and the application says so.

### Grammar

The [EBNF](../EBNF.md) does not yet hold sections, so the rules below add what this proposal needs and leave the rest as it is.

```ebnf
step             = { text item | ingredient | cookware | timer | product use }-,
                   [ white space, tail ], new line character, blank line ;
tail             = "->", { white space }, product, { white space, "+", white space, product } ;
product          = product name, [ "{", [ amount ], "}" ] ;
product name     = name start, { text item - "{" - "}" } ;
name start       = text item - "@" - "#" - "~" - "&" - "?" - "{" - "}" - white space ;

product use      = "@", [ optional marker ], "&", ( one word component | multiword component ) ;

section line     = "=", { "=" }, { white space }, [ section name ],
                   [ white space, "->", { white space }, product ], { white space }, { "=" } ;
```

Notes on the grammar:

- A `tail` is read only at the end of a step. Inside the tail, a line break counts as white space.
- A `product name` ends at ` + `, at `{`, or at the end of the tail, and white space at its ends is dropped.
- On a `section line` the trailing `=` signs are not part of the product's name.
- `amount` is the amount of an ingredient as the specification has it today.

### Scope

Not in this proposal:

- **`@&` on an ingredient.** cooklang-rs has an extension in which `@&flour{}` refers to an ingredient written earlier. This proposal defines `@&` for products only. Rule 5 keeps the two apart, so both can be in one file.
- **Where a step happens and what it does.** Containers, verbs and their classes would let an application remove the edges that rule 9 gets wrong.
- **How a result is divided between two products** with no written amount (rule 17).
- **Products in `.menu` files**, such as a Monday dinner whose leftovers are Wednesday's lunch.
- **A percentage of a product**, as in baker's percentages.
- **Products of cookware** (a lined tin) and of timers.
- **How an application draws the graph.**

### Backward compatibility

**Existing files.** No file found today changes its reading. About 16,700 recipe files were searched, most of them machine-written, and none holds `->` or `@&`. A hand-written file elsewhere may hold an arrow in a step:

| Written | Today | Proposed |
|---|---|---|
| `Turn the oven 180 -> 160 and bake.` | text | a product named "160 and bake.", which nothing uses, so it is listed as a by-product |
| `Heat 60->80 °C.` | text | text: no white space before the arrow |

Files written for the cooklang-rs extensions keep working: `@&flour{}` after `@flour{}` is still a reference to the ingredient, and the `@&(~1)name{}` forms are untouched.

**New files in applications that predate this proposal.** Run with CookCLI 0.38.1:

| Written | The step reads | The lists hold |
|---|---|---|
| `Mix @flour{200%g} and @water{120%ml} in a #bowl{}. -> shaggy dough` | "Mix flour and water in a bowl. -> shaggy dough" | flour and water, as they should |
| `Knead the @&shaggy dough{}.` | "Knead the &shaggy dough." | an ingredient "&shaggy dough" on the ingredient list and the shopping list |
| `Use half the @&dough{1/2}.` | "Use half the &dough." | "&dough 1/2" on both lists |
| `= Dough -> dough` | a section titled "Dough -> dough" | nothing |

The text stays readable. The shopping list gains one odd line per product, each starting with `&`. Nothing is reported as an error.

A parser built on cooklang-rs with its extensions on behaves differently. Run with cooklang-rs 0.19.0: each `@&name{}` with no ingredient of that name before it is the error "Reference not found", and the file does not parse. Applications that parse this way have to be updated before files with products are written for other people.

### Relation to other proposals

| Proposal | How they meet |
|---|---|
| [0018](0018-optional-components.md), optional components (accepted) | `@?&stock{}` is an optional use of a product |
| [0017](0017-yield-metadata.md), `yield` (accepted) | The dish's amount stays in front matter. Rule 15 is the same thing for a product on the way |
| [#162](https://github.com/cooklang/spec/pull/162), alternatives | A product is a component, so it can be an option: `@&stock{}\|@water{500%ml}`. Sections that are alternatives of each other may name the same product (rule 5), and should, so that `@&pastry{}` is found on every path. A label on the section line comes before the arrow: `= Pastry ?homemade -> pastry`. In a file with alternatives, "the step before it" and "the last step" in rules 7, 8, 9 and 19 are read on the chosen path |
| [#149](https://github.com/cooklang/spec/pull/149), component variables | Nothing in it names what a step makes. It splits a declared total of a bought ingredient between steps and checks that the parts add up, which is the check of rule 14 on a different thing |
| [#147](https://github.com/cooklang/spec/pull/147), baker's percentages | A percentage of a product becomes possible once products have names. Not defined here |

### Canonical cases

| Source | Reading |
|---|---|
| `Mix @flour{200%g} and @water{120%ml}. -> dough` | step text "Mix flour and water."; product "dough" |
| `Knead the @&dough{}.` after it | uses the whole dough; the name goes on; no "dough" on any list |
| `= Dough -> dough` and `== Dough -> dough ==` | a section named "Dough" with the product "dough" |
| `Rest the @&dough{}.` as the second step inside `= Dough -> dough` | the result of the step before it |
| `Add @&levain{100%g}.` | 100 g of the levain; the rest keeps the name |
| `Mash half: @&potatoes{1/2}.` then `@&potatoes{}` | a half, then the other half |
| `Separate @eggs{4}. -> yolks{4} + whites{4}` then `@&yolks{1}` | two products with counts; one yolk, three left |
| `Drain the @&pasta{}. -> pasta + pasta water{250%ml}` | the name "pasta" moves to the drained pasta; 250 ml of water named |
| `Simmer for ~{4%hours}. -> stock{2%l}` | product "stock", 2 l, which scales |
| `Bake.` then a line `-> loaf` in the same step | the tail on its own line; product "loaf" |
| `Heat 60 -> 80 °C, then strain. -> custard` | the first arrow is text; product "custard" |
| `Whisk. ->dough` and `Whisk.-> dough` | the first names "dough"; the second is text |
| `@&dough{}` with no product and no ingredient "dough" | reported; shown as "dough"; on no list |
| `-> butter` in a file with `@butter{50%g}` | reported: one name, two things |
| `Mix the @&wet mix{} with eggs. -> batter` then `@&wet mix{}` | reported: used up by the step that made the batter |
| `-> whites` that no step uses | a by-product, listed |

## Effect on applications which use Cooklang

### Specification updates

A section of the README for products, the grammar above in `EBNF.md`, and canonical tests. These follow acceptance.

### Display

A product in a step is shown by its name, and can link to the step or section that made it. The tail is not part of the step's text; an application may show it as a label on the step ("makes: batter"). By-products are listed with the recipe.

### Lists and scaling

Products are on no ingredient list and no shopping list, and are not matched against a pantry. A step's own list of what goes into it shows the product, marked as made in this recipe. Scaling follows rule 13.

### Diagnostics

- A use with no product of that name before it.
- A name used again after it was used up (rule 8).
- A product with the name of an ingredient or a cookware item.
- A name given twice outside the three cases of rule 5.
- A section line with a product and no steps.
- A section's own name used in its first step.
- Uses that add up to more than the amount made; a share above 1.
- A section product over a last step with two or more products.

A by-product is listed with the recipe, not reported as a fault.

### What a parser gives an application

For the recipe: the list of products, each with its name, the step or section that made it, the amount written, and its uses. For each step: the products it uses, with the amount or share, and the products it names.

### CookCLI (terminal and web-server)

`cook recipe` shows products in the step and keeps them off the ingredient list. `cook shopping-list` skips them. `cook doctor` gets the diagnostics. A graph or table output can follow ([cookcli#410](https://github.com/cooklang/cookcli/issues/410)).

### Mobile applications

A product in a step is drawn as a link to where it was made, and cook mode can show "from Shell" beside it. Products are not on the ingredient list or the shopping list.

### Editors, the language server and syntax highlighting

Highlighting for the tail and for `@&`. Completion of product names after `@&`, go to definition, and the diagnostics above. The language server parses with the cooklang-rs extensions on, so it refuses these files today.

### Converters and other writers

An application that writes recipes should write products only after the applications that read them have been released.

### Other parsers

Parsers that are not built on cooklang-rs follow the canonical tests. Until they do, they read the new forms as shown under [Backward compatibility](#backward-compatibility).

## Open questions

1. **Steps without a mark** (rule 9). The proposed rule gives every existing file a graph and makes naming the way to branch. Two other answers. A step always carries on from the one before it, named or not: right for "Cool to room temperature" after `-> custard`, wrong for a step that starts something new after `-> seasoned steaks`. Or only written uses make edges, and a file without marks has no graph. In 27 test recipes written with these forms, 35 steps follow a step that named a product; 28 of them use a product by name, so the rule decides nothing there. Of the other 7 the proposed rule is right in 4, wrong in 1, and 2 can be read either way.
2. **Named before use** (rule 6). It matches cooking order and keeps the graph free of cycles. Finding a name anywhere in the file would let an author write the main part first and the sauce below it, and would need a cycle check.
3. **Names for the whole file** (rule 6). The alternative keeps a step's product inside its section and lets only section products cross sections.
4. **A name goes on** (rule 8). The alternative is that every use ends the name, and each step that works on the dough names it again.
5. **A number with no unit** (rule 11) is a share, or a count when the product was named with a count. The alternative is always a share, with counts written with a unit word.
6. **One set of names** for products, ingredients and cookware (rule 5). It costs a rename now and then: a pie with a whipped cream made in the recipe and a bought whipped cream as garnish. Cookware could be left out of the set.
7. **An arrow in prose** (rule 1). Every arrow that ends a step starts a tail. An arrow could instead be text when what follows it ends in a full stop, which saves the rare prose arrow and makes the reading depend on punctuation.
8. **The dish** (rule 19) is what the last step makes. It could instead be the product of the last section line that has one.
9. **`@&` on an ingredient.** Whether the cooklang-rs reference to an earlier ingredient should enter the specification, and whether it adds to the amount or takes from a declared total, is left for a separate proposal.
10. **Canonical tests.** The tests need a shape for a product, a use and the graph. This proposal defines the syntax and its meaning, not the parsed model.

## Alternatives considered

1. **Counting steps**, the cooklang-rs form `@&(~1)dough{}`. It is exact and it is the only form with users today. Its costs are in [Motivation](#motivation). It can stay an extension that fills the same model.
2. **`-> @dough`**, the arrow with an ingredient token. An application that predates this proposal would read `@dough` as an ingredient and put it on the shopping list. A bare name is text to it.
3. **A bare `&dough`** for the use. An ampersand in prose ("salt & pepper") becomes a hazard for the parser.
4. **The section's name is the product**, with no arrow. No new token, but section names are titles ("Shaping and proof"), and a step could not make a product.
5. **Names declared in front matter.** The declaration sits far from the step that makes the thing, and an older application shows nothing where the cook reads.
6. **Two arrows for two products** (`-> yolks -> whites`), or brackets (`-> (yolks, whites)`). The `+` reads as it does in a sentence and needs no extra marks.
7. **A mark for by-products** (`-> whites [keep]`). Rule 20 gives the same answer with nothing to write.

## Acknowledgments

Thanks to the people who asked for this in the discussions linked above; to the authors of cooklang-rs for the intermediate preparations extension, which showed that a product has to stay off the lists; to the author of [tabular-recipes](https://github.com/johnhkchen/tabular-recipes), whose write-up lists exactly what counting steps makes hard; and to the authors of [Gram](https://github.com/gram-lang/gram), which names its intermediates and refers to them by name.
