# Units on the shopping list and in recipe references

* Proposal: [0028-shopping-list-units](0028-shopping-list-units.md)
* Authors: [Alexey Dubovskoy](https://github.com/dubadub)
* Review Manager: TBD
* Status: **Awaiting review**
* Related: [0016 shopping list format](0016-shopping-list-format.md), [0017 yield](0017-yield-metadata.md), [unit system metadata (PR #148)](https://github.com/cooklang/spec/pull/148), [alternative units (PR #157)](https://github.com/cooklang/spec/pull/157), [optional `%` (PR #161)](https://github.com/cooklang/spec/pull/161)

## Introduction

A shopping list built from several recipes shows an ingredient once per unit: `milk 1 cup, 200 ml`. This proposal adds an opt-in setup that makes a tool show one amount per ingredient. Amounts of the same kind (cups and millilitres, grams and pounds) are added through a unit table. Weight and volume are added only when a density is known for the ingredient, and such a total is marked as approximate. No recipe syntax changes, and a recipe is still shown in the units its author wrote.

The same comparison decides whether a recipe reference can be scaled. `@./Stock{1%l}` against `yield: 500%ml` is an error today because the unit text differs. The second part of the proposal defines how a reference's unit is matched to a `yield`.

Discussion thread: [Support for Unit and Serving Size Conversions](https://github.com/cooklang/spec/discussions/142). Related: [cookcli#673](https://github.com/cooklang/cookcli/issues/673) (settings for unit conversion), [cookcli#199](https://github.com/cooklang/cookcli/issues/199) (pantry amounts are subtracted only when the unit is written the same way).

## Motivation

### The list repeats an ingredient once per unit

Two recipes, one American and one European:

```cooklang
Add @milk{1%cup}, @butter{100%g}, @flour{2%cups}, @sugar{1%lb}, @oil{3%tbsp} and @onion{1}.
```

```cooklang
Add @milk{200%ml}, @butter{2%tbsp}, @flour{250%g}, @sugar{100%g}, @oil{50%ml} and @onion{150%g}.
```

`cook shopping-list` on the two files (CookCLI 0.38.1):

```
milk   1 cup, 200 ml
butter 100 g, 2 tbsp
flour  2 cups, 250 g
sugar  100 g, 1 lb
oil    50 ml, 3 tbsp
onion  1, 150 g
```

Nobody can shop from "1 cup, 200 ml" without doing the sum. The same unit spelled two ways is not added either: `1%cup` and `2%cups` give `1 cup, 2 cups`, `1%tbsp` and `2%tablespoons` give `2 tablespoons, 1 tbsp`, and `1%Tbsp` and `1%tbsp` give `1 Tbsp, 1 tbsp`.

There are three separate problems:

1. **The same unit, spelled differently** (`cup` and `cups`, `tbsp` and `tablespoons`).
2. **The same kind of unit, different units** (cup and ml, g and lb).
3. **Weight in one recipe, volume in another** (butter, flour).

A unit table solves the first two. The third needs the ingredient's density.

How often each happens was measured on a collection of about 16,000 recipes, in 20,000 random lists of five recipes, counting the ingredients that two or more of the five measure by weight or volume:

| The recipes that share the ingredient use | Share | One amount on the CookCLI list today |
|---|---|---|
| the same unit, same spelling | 23% | yes |
| the same unit, spelled differently | 37% | no |
| different units of the same kind | 33% | no |
| weight in one, volume in another | 7% | no |

Three shared ingredients in four come out as two or more amounts. That collection was written with a language model and is mostly in US volume units, so 7% is a low figure for weight against volume. A cookbook that mixes American and European sources crosses that line far more often.

### Tools give different lists from the same files

- **CookCLI** was built with the unit table of [cooklang-rs](https://github.com/cooklang/cooklang-rs) until [cookcli#433](https://github.com/cooklang/cookcli/pull/433). It now adds amounts only when the unit text is identical, as shown above.
- **cooklang-rs with its bundled unit table**, which other applications are built with, adds units of the same kind into the unit of whichever recipe comes first. The two recipes above give `milk 1.845 c` in one order and `milk 436.588 ml` in the other, and `sugar 1.22 lb` or `553.592 g`. Butter and flour stay two amounts.
- **The mobile bindings** in cooklang-rs combine amounts by unit text, with no unit table. (Read in the source, not run on a device.)

The unit table was turned off in CookCLI for a good reason. In [cookcli#432](https://github.com/cooklang/cookcli/issues/432) a recipe's `@cocoa{5/8%cup}` was shown as `10 tbsp`, and `1 5/8 cup` became `1 2/3`. A recipe should show what its author wrote. Whatever converts list totals must not bring conversion back into the recipe view.

### The pantry and recipe references compare unit text too

A pantry entry `flour = "5%kg"` is not subtracted from `@flour{500%g}`, because `kg` is not `g` (cookcli#199).

A reference is scaled by a unit only when the unit text equals the unit text of the `yield`. With `Stock.cook` declaring `yield: 500%ml` and `Bechamel.cook` declaring `yield: [500%ml, 350%g]` (CookCLI 0.38.1):

| Reference | Result |
|---|---|
| `@./Stock{250%ml}` | Scaled by 0.5 |
| `@./Stock{1%l}`, `@./Stock{1%cup}` | Not scaled: "its yield is in ml, so ask for it in ml". The shopping list shows `Stock 1 l` instead of the stock's ingredients |
| `@./Stock{250%ML}` | Not scaled, although 0017 says the match ignores case |
| `@./Bechamel{250%ml}`, `@./Bechamel{175%g}` | Not scaled: the list form of `yield` is not read |

An author who writes a litre of stock against a recipe that yields 500 ml has said something clear. It should not be an error.

## Proposed solution

### One amount per ingredient, on request

A setup file turns it on and names the unit system for totals:

```toml
[shopping]
units = "metric"
```

With it, the two recipes above give:

```
milk    437 ml
butter  ≈ 127 g
flour   ≈ 501 g
sugar   554 g
oil     95 ml
onion   150 g, 1
```

- Milk, sugar and oil are sums through the unit table. They are exact, apart from being rounded up.
- Butter and flour cross between weight and volume. That needs a density (0.911 g/ml for butter and 0.53 for flour in this example), the total is marked `≈`, and the amounts as written stay available: `250 g + 2 cups`.
- The onion has an amount by weight and a count. They are listed side by side, as today.
- With no density for flour, the list says `flour 474 ml, 250 g`. A density is never assumed.
- The list is the same whichever recipe comes first.

Without the setup file, list totals stay as they are today.

### A reference is matched by kind

`@./Stock{1%l}` against `yield: 500%ml` scales the stock by 2. `@./Bechamel{6%oz}` against `yield: [500%ml, 350%g]` is measured against the `350%g` entry. Nothing that is shown is converted: the reference still reads `1 l`. This part needs no setup.

## Detailed design

### The setup file

`units.conf`, a TOML file kept where the tool keeps the aisle and pantry configuration (the `config` folder in CookCLI and the apps).

```toml
[shopping]
units = "metric"          # or "us customary"

[density]                 # grams per millilitre
"bread flour" = 0.55
"oat milk" = { density = 1.03, buy = "volume" }
```

- **`units`** names the system that totals are shown in: `metric` or `us customary`, the values of the `unit system` key in PR #148. When the key or the file is missing, totals are not converged.
- **`[density]`** is optional. Each entry gives a density for an ingredient name and overrides the built-in table (below). A plain number means the ingredient is bought by weight. `buy = "volume"` says it is bought by volume. The two forms follow the pantry file, where a value is a string or a table.
- **`volume`** and **`mass`** in `[shopping]` are optional lists of units that replace the default units for totals: `volume = ["fl oz", "pt", "qt"]`.
- An application may offer the same choice as a setting (As written, Metric, US). The file is what moves between devices with the recipes. A tool that does not know the file ignores it.

### The unit table

The rule needs to know which unit names are the same unit, and how units of one kind relate. This is the volume and mass part of the table that cooklang-rs bundles.

| Kind | Unit | Read from | Shown as | In the base unit |
|---|---|---|---|---|
| volume | millilitre | `ml`, `mL`, `milliliter`, `millilitre` and plurals | `ml` | 1 ml |
| volume | litre | `l`, `L`, `liter`, `litre` and plurals | `l` | 1000 ml |
| volume | teaspoon | `tsp`, `tsp.`, `teaspoon`, `teaspoons` | `tsp` | 4.928921 ml |
| volume | tablespoon | `tbsp`, `tbsp.`, `tbs`, `tbs.`, `tablespoon`, `tablespoons` | `tbsp` | 14.786764 ml |
| volume | fluid ounce | `fl oz`, `fl. oz.`, `fluid ounce`, `fluid ounces` | `fl oz` | 29.573529 ml |
| volume | cup | `cup`, `cups`, `c` | `cup` | 236.588236 ml |
| volume | pint | `pt`, `pint`, `pints` | `pt` | 473.176473 ml |
| volume | quart | `qt`, `quart`, `quarts` | `qt` | 946.352946 ml |
| volume | gallon | `gal`, `gallon`, `gallons` | `gal` | 3785.411784 ml |
| mass | gram | `g`, `gram`, `grams` | `g` | 1 g |
| mass | kilogram | `kg`, `kilogram`, `kilograms` | `kg` | 1000 g |
| mass | ounce | `oz`, `oz.`, `ounce`, `ounces` | `oz` | 28.349523 g |
| mass | pound | `lb`, `lb.`, `lbs`, `pound`, `pounds` | `lb` | 453.59237 g |

Other SI prefixes of the litre and the gram (`cl`, `dl`, `mg`) are read the same way.

- **Case.** A unit is looked up as written, then in lower case, so `Tbsp`, `Cup`, `Kg` and `ML` are found. The temperature symbols `C` and `F` are the exception: they are never looked up in lower case, so `1%C` is not a cup. In the collection measured above `Tbsp` is written 2,132 times and `Cup` 167 times, of about 203,000 amounts with a unit.
- **`lbs`** is new. It is not in the cooklang-rs table today and is written 592 times in the same collection.
- **The volume units are US customary.** A US cup is 236.588 ml, and a metric cup is 250 ml. When a recipe declares `unit system` (PR #148), its amounts are read in that system by a tool that has the units for it. Without the key, the values above apply.
- **Default units for totals.** Metric: `ml`, `l` and `g`, `kg`. US customary: `tsp`, `tbsp`, `cup` and `oz`, `lb`.
- A tool may know more units and more kinds. Amounts of another kind (length, for `@ginger{2%cm}`) may be added among themselves in the same way. They never merge with weight or volume.

### Totals on the list

For each ingredient name, after scaling and after synonyms are folded into one name:

1. **Sort the amounts by kind:** weight, volume, and everything else. "Everything else" is an amount with no unit (`@onion{1}`), a unit the table does not know (`clove`, `can`, `pinch`), or text (`a splash`).
2. **Add each kind in its base unit:** all weights in grams, all volumes in millilitres. The result does not depend on the order of the recipes.
3. **Cross only when it merges something.** If the ingredient has both a weight total and a volume total and a density is known, convert the side it is not bought by into the side it is bought by, and add. If no density is known, keep both totals.
4. **Pick the unit from the total:** the largest of the system's default units in which the total is 1 or more, or the smallest when there is none. 437 ml, 1.25 kg, 2 cups. When rounding (next step) lifts the total to 1 of the next larger default unit, that unit is used: 999.4 g is `1 kg`, not `1000 g`.
5. **Round up, in the unit's own step.** Whole numbers for g and ml, two decimals for kg and l, quarters for US customary units. Always up and never to the nearest, so the list does not ask for less than the recipes need. A total that passes a step by less than a thousandth of the step is that step, so 250 g and 250 g are 500 g whatever the arithmetic leaves behind.
6. **Mark a crossed total as approximate,** with `≈` in text and a flag in structured output, and keep the amounts as written with it. A total that used only the unit table carries no mark.
7. **List "everything else" after the total, unchanged:** `onion 150 g, 1`.

An ingredient that has only weights, or only volumes, is never crossed, whatever the density table says. If every recipe measures flour in cups, the list says cups, or millilitres in metric. Crossing adds an error, so it is used only where it removes a line.

### The density table

A density is looked up by ingredient name, first in `[density]` of the setup file and then in a built-in table.

- **Names** are compared in lower case, with hyphens read as spaces, after synonyms from the aisle configuration are folded (`flour|plain flour|all-purpose flour`). The match is exact. `unsalted butter` does not find `butter` unless a row lists it, so each row of the built-in table carries a list of names, as a unit does.
- **No hit means no crossing.** No default density is assumed: not 1.0, and not "it is a liquid, so about the same as water".
- **The built-in table holds only ingredients for which one number is honest:** liquids, fats, sugars, flours, grains, syrups. Milk, cream, oils, butter, honey, flour, sugar, rice. Each row has a density, the side the ingredient is bought by, and its names.
- **It leaves out ingredients whose cup weight depends on the knife:** shredded cheese, chopped onion, spinach leaves, sliced mushrooms. One density for them would be wrong by a factor of two. It also leaves out plain `salt`, because table salt and flaked salt differ by about that much per spoon. These stay two amounts unless the author gives the equivalent (PR #157).
- **Source.** Values are taken from the portion weights in USDA FoodData Central and checked by hand. A first table of about 150 rows is realistic. It would be published with the conventions as a data file, so that tools agree on totals. The table is not part of this pull request, and the densities used in this document are illustrative.

How wrong a crossed total can be: a US cup of flour weighs 120 to 150 g depending on how it is filled, so `2 cups` is 240 to 300 g. That is why the total is marked and why the rule crosses only when it has to.

### Where the rule applies

- **The recipe view** and cook mode are not touched. Neither is the ingredient list of a single recipe, where `@butter{100%g}` and `@butter{2%tbsp}` in two steps stay as written. The rule is for totals across recipes.
- **Menus.** A total for a `.menu` file is a total across recipes and follows the rule.
- **The pantry.** Pantry amounts go through the same rule before they are subtracted, so `flour = "5%kg"` covers `@flour{2%cups}` when a density is known, and covers `@flour{500%g}` with the unit table alone. A remainder that crossed is marked `≈`. With no density the pantry amount is not subtracted, as today.
- **`.shopping-list` files** keep amounts in the units they were written in. A tool that writes accepted optional amounts into the file writes them unconverted. Totals are converged when the list is shown. "Compatible units" in the rules for [optional ingredients](../conventions.md#optional-ingredients) on a shopping list means units of the same kind in this table.
- **Alternative units.** If PR #157 is accepted and a recipe gives `@flour{2%cups|260%g}`, the amount in the kind the list needs is taken as the author wrote it. The author's figure is used before the table, and the total is not marked on its account.

### Recipe references and `yield`

This replaces "Only matching units are supported" in [Scaling Referenced Recipes](../conventions.md#scaling-referenced-recipes) and step 2 of the scaling rule in [0017](0017-yield-metadata.md), "find the entry whose unit matches the reference's unit".

The reference's amount is compared with each entry of the referenced recipe's `yield`. The first of these that applies gives the factor:

1. **The same unit,** found through the unit table, so `ML`, `ml` and `millilitres` are one unit. The factor is the requested amount divided by the yield.
2. **The same kind.** Both are converted to the base unit. `{1%l}` against `500%ml` is 2, and `{1%cup}` is 0.473.
3. **Several yields: the entry of the same kind is used.** With `yield: [500%ml, 350%g]`, `{6%oz}` is a weight and is measured against `350%g` (0.486), and `{1%cup}` against `500%ml`. No density is worked out from the two entries. This is the way to make a recipe referable by weight and by volume.
4. **The other kind, with a density the user wrote.** The recipe has one yield and the reference is of the other kind. If `[density]` in the setup file has an entry under the recipe's name (the file name without its extension, in lower case), it is used. With `stock = 1.0`, `{250%g}` against `yield: 500%ml` is ≈ 0.5. The built-in density table is not used here.
5. **Otherwise** it is an error, as today, and the message says what to add: "Stock yields a volume and no weight is known for it. Add a second yield, such as `yield: [500%ml, 500%g]`."

- Rules 1 to 3 are exact, need no setup file, and are always on. They turn an error into a result and change no unit anyone sees.
- A factor from rule 4 is approximate. The tool says so where it shows the factor, and every list total that includes an amount scaled by it is marked `≈`.
- A unit the table does not know (`yield: 6%balls`) is matched by its text, ignoring case, as 0017 says. `{1%ball}` against `6%balls` is still an error.
- The rule is the same in recipes, `.menu` files and `.shopping-list` files.
- When an application shows the scaled recipe's yield, every entry is multiplied by the factor and keeps its unit: `[500%ml, 350%g]` at 0.486 is 243 ml and 170 g.
- A missing `yield` is not worked out from the ingredients. Cooking changes weight, and a stock does not weigh what went into the pot.

### Backward compatibility

- No recipe syntax changes, and no existing file changes meaning.
- A tool that does not know `units.conf` ignores it and shows the list as today. Two devices may show different lists until both are updated.
- A reference such as `@./Stock{1%l}` against `yield: 500%ml` is scaled by an updated tool and is an error in an older one. That is what the older tool does today, so nothing gets worse, but an author who writes such references should know that older applications will not expand them.
- With no setup file, an application that adds amounts through the unit table today should stop doing so, so that all tools show the same list (see the open questions).

### Canonical cases

List totals, for the two recipes in the motivation unless other amounts are given. Densities: butter 0.911, flour 0.53.

| Case | Setup and amounts | Expected |
|---|---|---|
| No setup | no `units.conf` | amounts as written, as today |
| Same kind | `metric` | `milk 437 ml`, `sugar 554 g`, `oil 95 ml`, no mark |
| Spelling | `metric`; `1%cup` + `2%cups`; `1%tbsp` + `2%tablespoons` | `710 ml`; `45 ml` |
| Case | `metric`; `1%Tbsp` + `1%tbsp` | `30 ml`; `1%C` is not a cup |
| Order | `metric`, recipes swapped | the same list |
| Crossing, bought by weight | `metric` | `butter ≈ 127 g`, `flour ≈ 501 g`, written amounts kept |
| Crossing, bought by volume | `metric`; milk `1%cup` + `200%g`, density 1.03, `buy = "volume"` | `milk ≈ 431 ml` |
| No density | `metric`; no density for flour | `flour 474 ml, 250 g`, no mark |
| One kind only | `metric`; flour `2%cups` in both recipes, density known | `flour 947 ml`, not grams |
| Count and unknown units | `@onion{1}`, `@onion{150%g}`, `@garlic{2%cloves}` | `onion 150 g, 1`; `garlic 2 cloves` |
| Text amount | `@oil{a splash}` + `@oil{50%ml}` | `oil 50 ml, a splash` |
| Larger unit | flour `600%g` + `650%g` | `flour 1.25 kg` |
| Exact sum | `250%g` + `250%g` | `500 g`, not `501 g` |
| US customary | `us customary` | `sugar 1 1/4 lb` (1.22 lb), `milk 2 cups` (1.845 cups) |
| US customary, exact sum | `us customary`; `1%cup` + `2%cups` | `3 cups`, not `3 1/4 cups` |
| Next unit | `metric`; `999.4%g` in total | `1 kg` |
| Override | `[density]` has `flour = 0.60` | flour uses 0.60 |
| Synonym | aisle configuration has `flour\|plain flour`; recipes use both | one line, density looked up as `flour` |
| Pantry | pantry has `flour = "5%kg"`; recipes need `2%cups` + `250%g` | flour is not on the list |
| Recipe view | setup present | the recipe still shows `1 cup`, `2 cups`, `1 lb` |

References. `Stock.cook` has `yield: 500%ml`, `Bechamel.cook` has `yield: [500%ml, 350%g]`, `Rolls.cook` has `yield: 6%balls`.

| Reference | Expected |
|---|---|
| `@./Stock{250%ml}` | 0.5 |
| `@./Stock{1%l}` | 2 |
| `@./Stock{1%cup}` | 0.473 |
| `@./Stock{250%ML}` | 0.5 |
| `@./Stock{250%g}` | error; the message names the missing weight yield |
| `@./Stock{250%g}` with `stock = 1.0` in `[density]` | ≈ 0.5, marked |
| `@./Bechamel{250%ml}` | 0.5 |
| `@./Bechamel{175%g}` | 0.5 |
| `@./Bechamel{1%cup}` | 0.473, from the `500%ml` entry |
| `@./Bechamel{6%oz}` | 0.486, from the `350%g` entry, no mark |
| `@./Rolls{3%balls}`, `@./Rolls{3%Balls}` | 0.5 |
| `@./Rolls{1%ball}` | error |
| `@./Stock{1%l}` with no `units.conf` | 2 |

## Effect on applications which use Cooklang

### Parsers and libraries

1. One function for list totals. It takes the amounts per ingredient, the unit table, the setup and the density table, and returns for each name the total, whether it is approximate, the amounts as written, and the rest that could not be added.
2. One function for references. It takes the reference's amount, the referenced recipe's `yield` (one value or a list) and its name, and returns the factor and whether it is approximate, or the reason there is none. It needs the unit table even in a tool that parses recipes without one.
3. `yield` is read as a list as well as a single value. In cooklang-rs it also stops being read as `servings`: today every `yield` with a unit prints "Unsupported value for key: 'yield'".
4. In cooklang-rs: the lower-case lookup, `lbs`, a reader for `units.conf`, and sums in the base unit in place of the first amount's unit.

### CookCLI (terminal and web-server)

1. Recipes keep being parsed without unit conversion, so they show what the author wrote. cookcli#433 stands.
2. `cook shopping-list`, the pantry commands and the web UI build totals with the rule when `units.conf` asks for it. A `--units` option overrides the file for one run.
3. The `≈` mark in text output and an `approximate` field in JSON; the written amounts behind a total with `--verbose` and in the web UI.
4. References are scaled by the matching rules in `cook recipe`, the shopping list, menus and the web UI. The error text says which yield to add.
5. `cook doctor` can list the ingredients written by weight in some recipes and by volume in others that have no density.

### Mobile applications

1. A Units setting for the shopping list (As written, Metric, US), stored in `units.conf` in the synced folder.
2. A crossed total shows `≈`. Tapping it shows the amounts as written.
3. The pantry subtracts across units.
4. References with a unit of another kind are expanded where they were skipped before.

### Editors and other applications

1. Applications built with the cooklang-rs unit table use the new function and, with no setup, stop adding into the first recipe's unit.
2. No change to syntax highlighting or to the language server.

## Open questions

1. **Opt-in or on by default.** The proposal is opt-in, because cookcli#433 is recent and people expect to see what was written. One part could be on for everyone: adding two spellings of the same unit (`1 cup` and `2 cups`) converts nothing, and it is 37% of shared ingredients in the measurement above.
2. **A unit table in the conventions.** PR #148 leaves conversion tables to each application. This proposal needs tools to agree on which names are the same unit and on the ratios, or two tools give different totals from the same files. The table above is the minimum for that. Rounding and the default units for totals could be left to applications instead of being fixed here.
3. **US customary and British imperial.** The table has US volumes. An imperial pint or fluid ounce differs, and a metric cup is 250 ml. Whether the table grows a second set of volume units depends on PR #148.
4. **What a total is shown in when weight meets volume.** Here: per ingredient, by weight unless the table says volume. The alternatives are always weight, or the side that holds the larger share of the total.
5. **Where densities are kept.** Here: a small built-in table published with the conventions, and `[density]` in `units.conf` on top. If an ingredient database with other properties is specified later, it becomes the main home and `[density]` remains a short override.
6. **Rule 4 for references.** It is the only matching rule that is approximate, and the only one that needs the flag carried from scaling into list totals. Dropping it loses little, because a second `yield` entry already covers the author who knows the weight.
7. **Applications that add units today.** With no setup they would go back to amounts as written, which removes a sum some people may rely on.
8. **Steps for US customary totals.** Quarters for every unit is simple. Eighths for teaspoons, or whole ounces, may read better.

## Alternatives considered

1. **Convert everything to grams.** [Gram](https://github.com/gram-lang/gram), another recipe markup language, merges amounts only through mass. Its analyzer gives every amount a mass from a per-ingredient density or unit weight, adds them when every amount got one, and flags the total as an estimate when a database density was used. Otherwise it keeps the amounts apart and shows them together as mixed units. It never assumes a density. This proposal takes the estimate flag, the fallback to separate amounts and the rule against guessing from it. It differs in two ways. Cups and millilitres need no ingredient data at all, only a unit table. And grams are right for pastry but wrong for a list: nobody buys 450 g of milk.
2. **Assume a density when none is known,** such as that of water. A cup of flour is not 240 g. A wrong total with no warning is worse than two amounts.
3. **Ask a nutrition service for densities when the list is built.** The list would then need a network, and two tools would agree only while they use the same service. Such data is a good source for building the table.
4. **Always convert to the side the ingredient is bought by,** also when all recipes agree on the kind. It is more uniform, and it makes every such line approximate with nothing gained.
5. **A density override in a recipe's front matter.** An author who knows the weight can write it as an alternative amount (PR #157), which is more direct.
6. **Only an application setting,** with no file. It does not travel with the recipes, and the densities still need a home.
7. **Work out a missing weight yield from the ingredients.** Cooking changes weight. A declared second yield is exact, and it is already in the conventions.

## Not in this proposal

- Count against weight (`1 onion` and `150 g`, "1 can = 400 g").
- Colloquial and localised units (`knob`, `tasse`, `ст. ложка`). Once they are in a unit table they converge with no further rule.
- Pack sizes ("butter comes in 250 g blocks") and waste ("buy 285 g of avocado for 200 g in the bowl").
- Density by preparation (sifted flour, packed brown sugar). One number per name.
- Showing a recipe in the reader's units: PR #148 and PR #157.

## Acknowledgments

Thanks to [@RPGillespie6](https://github.com/RPGillespie6) for opening the discussion on unit conversion and to [@stevenbell](https://github.com/stevenbell) for the case of scaled amounts that should move to a larger unit; to [@Zheoni](https://github.com/Zheoni), whose unit table in cooklang-rs this builds on; to [@LukaHummel](https://github.com/LukaHummel) for the `unit system` proposal; and to the Gram project, whose handling of density is the prior art for the weight and volume part.
