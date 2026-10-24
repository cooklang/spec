# Hidden ingredients and cookware

* Proposal: [0020-hidden-components](0020-hidden-components.md)
* Authors: [Alexey Dubovskoy](https://github.com/dubadub)
* Review Manager: TBD
* Status: **Awaiting review**
* Previous Proposal: [0018-optional-components](0018-optional-components.md)

## Introduction

This proposal moves the hidden marker `-` from a parser extension into the Cooklang specification. Writing `@-water{3%l}` or `#-paper towel{}` marks that ingredient or cookware as hidden: it is highlighted and scaled inside the step like any other component, but it never appears in the recipe's ingredient or cookware list, on a shopping list, or in pantry matching. The syntax is already implemented in [cooklang-rs](https://github.com/cooklang/cooklang-rs/blob/main/extensions.md#modifiers) as part of the "component modifiers" extension; this proposal standardises the `-` modifier only, defines what applications should do with it, and specifies how it combines with the optional marker `?` from [proposal 0018](0018-optional-components.md).

Discussion threads: [Syntax for cooking methods](https://github.com/cooklang/spec/discussions/127) (where an author writes `-@Cold water{to cover}` to keep water off the list), [hidden ingredients are rendered in the ingredient list in the playground](https://github.com/cooklang/cooklang-rs/issues/66).

## Motivation

Many things a recipe tells you to add are not things you buy:

- water to boil the pasta, to cover the bones, to deglaze a pan;
- oil or butter to grease a tin;
- "salt to taste" and "a pinch of sugar" that any kitchen has;
- a reserved cup of pasta water, or the brine from a jar you already opened.

Authors still want to write these as ingredients. Inside a step an ingredient is highlighted, carries a quantity that scales with the recipe, can take a preparation note, and is visible to tools that analyse steps. A stock recipe that says "simmer in `@water{3%l}`" needs the 3 litres to double when the recipe is doubled.

Today there are two ways to write it, and both are wrong:

```cooklang
Simmer the bones in water{3%l}.      -- plain text: no highlight, does not scale
Simmer the bones in @water{3%l}.     -- ingredient: 3 l of water lands on the shopping list
```

The first loses everything the markup gives. The second puts junk on the shopping list, makes "what can I cook with what I have" complain that water is missing from the pantry, and inflates the ingredient list with items nobody reads.

The cooklang-rs parser has supported `@-water` for a long time behind the `COMPONENT_MODIFIERS` extension flag, and authors have adopted it. Because it is an extension and not part of the spec, support is inconsistent:

- Tools that enable the extension (cooklang-chef, the cookbook generator, the cook.md importer) treat `@-water` as hidden water.
- Tools that follow the spec strictly (CookCLI, the mobile apps) parse the same text as a required ingredient literally named `-water`, and put `-water` on the shopping list.

The same recipe therefore looks correct in one tool and corrupted in another. This is the same situation that led to the `?` marker being standardised in proposal 0018, and the fix is the same: define the `-` marker in the spec so every implementation can support it unconditionally, without pulling in the rest of the modifiers extension.

## Proposed solution

A `-` placed immediately after the `@` or `#` sigil marks that ingredient or cookware as hidden:

```cooklang
---
yield: 2%l
---

Roast @chicken carcasses{2} on a #roasting tray{} at 220 °C for ~oven{30%minutes}.

Simmer the bones in a #stockpot{} with @-water{3%l}, @onion{1}(halved), @carrot{2} and @bay leaves{2} for ~{3%hours}.

Strain through a #fine sieve{} lined with a #-paper towel{}. Season with @-salt to taste.
```

The parsed recipe has:

| Ingredient | Quantity | Hidden |
| --- | --- | --- |
| chicken carcasses | 2 | no |
| water | 3 l | **yes** |
| onion | 1 | no |
| carrot | 2 | no |
| bay leaves | 2 | no |
| salt | | **yes** |

and cookware `paper towel` marked hidden.

Inside the steps every one of these renders as an ingredient or cookware item: highlighted, with its quantity, and scaled with the recipe (doubling the stock asks for 6 l of water). The ingredient list, however, shows only chicken carcasses, onion, carrot and bay leaves; the cookware list shows the roasting tray, stockpot and sieve. Water, salt and the paper towel are nowhere but in the steps.

The `-` is not part of the name. It works with every existing form of ingredient and cookware: single word (`@-salt`), multi-word (`@-neutral oil{}`), with quantity (`@-water{3%l}`), with a preparation note (`@-water{3%l}(cold)`), and recipe references (`@-./sides/steamed rice{}`).

Compared with writing the item as plain text, the marker keeps highlighting, scaling and notes. Compared with writing a normal ingredient, it keeps the shopping list and pantry honest.

## Detailed design

### Grammar

The `ingredient` and `cookware` productions, which after proposal 0018 take an optional marker directly after the sigil, now take a small set of markers:

```ebnf
ingredient           = one word ingredient | multiword ingredient ;
one word ingredient  = "@", [ markers ], one word component ;
multiword ingredient = "@", [ markers ], multiword component ;

cookware             = one word cookware | multiword cookware ;
one word cookware    = "#", [ markers ], one word component ;
multiword cookware   = "#", [ markers ], multiword component ;

(* each marker at most once, in either order; none is part of the name *)
markers              = optional marker
                     | hidden marker
                     | optional marker, hidden marker
                     | hidden marker, optional marker ;
optional marker      = "?" ;
hidden marker        = "-" ;
```

Rules:

1. The marker must follow the sigil immediately, or immediately follow the other marker. `@- water` is not a hidden ingredient; per existing rules a sigil followed by whitespace is plain text. `@-{3%l}` is plain text too, because there is no name.
2. Each marker is recognised at most once. In `@--salt` the first `-` is the marker and the name is `-salt`, mirroring the `@??thyme` rule of proposal 0018.
3. The two markers may be combined in either order: `@?-chives` and `@-?chives` both describe an ingredient named `chives` that is optional and hidden.
4. Everything after the markers follows the existing naming rules unchanged. A `-` anywhere else is ordinary text or part of a name: `@all-purpose flour{}`, `@salt{} - to taste` and the `--` comment syntax are unaffected.
5. Timers are unaffected. `~-{5%minutes}` has no special meaning.

### Parsed model

Parsers must expose a boolean `hidden` property on every ingredient and cookware item, defaulting to `false`, alongside the `optional` property of proposal 0018. The name must not include the marker.

In the canonical test format a hidden ingredient looks like:

```yaml
testHiddenIngredient:
  source: |
    Simmer in @-water{3%l}
  result:
    steps:
      -
        - type: text
          value: "Simmer in "
        - type: ingredient
          name: "water"
          quantity: 3
          units: "l"
          hidden: true
    metadata: {}
```

Existing tests are unaffected because `hidden` defaults to `false` and may be omitted when false. Canonical tests to add with the docs change: a hidden single-word ingredient, a hidden multi-word ingredient with quantity, hidden cookware, a hidden and optional ingredient in both marker orders, `@--salt` (marker once, name `-salt`), `@- water` (plain text), and a hyphen inside a name (`@all-purpose flour{}`, unchanged).

### Hiding is per occurrence

As with optionality, the flag belongs to one occurrence of a component in the text, not to the ingredient as a whole. The same ingredient can be listed in one place and hidden in another:

```cooklang
Rub the pork with @salt{1%tbsp} and leave overnight.

Boil the potatoes in @-water{} with a little @-salt.
```

The ingredient list shows `salt 1 tbsp`. The pinch for the water is in the step and nowhere else.

### Aggregation

When building a recipe's ingredient list, hidden occurrences are skipped entirely: they are not listed on their own and their quantities are not added to a listed occurrence of the same name. In the example above the list says `salt 1 tbsp`, not `salt 1 tbsp + some`. The same applies to cookware.

This is the rule cooklang-rs already applies through its `should_be_listed` check.

### Scaling and conversion

Hidden components scale and convert exactly like listed ones. The inline rendering of `@-water{3%l}` in a doubled recipe is 6 l. Hiding changes where a component is shown, not what its quantity means.

### Hidden and optional together

`@-?chives` is an ingredient that is both hidden and optional. Hidden decides listing, optional decides whether the recipe works without it, and the two do not interact:

- it is not shown in the ingredient list and is never offered for a shopping list, because hidden items are not listed at all;
- inside the step it may be marked as optional, as any optional component.

The combination is rare in practice but is accepted so that recipes written for cooklang-rs, which already allows it, parse the same way everywhere.

### Recipe references

A hidden recipe reference, `@-./sides/steamed rice{}`, keeps the link but contributes nothing to lists. Applications still render it inline as a reference (with navigation to the referenced recipe and its multiplier), but when they expand references into ingredients, for the ingredient list, a shopping list or pantry matching, a hidden reference is skipped as a whole. This lets a recipe suggest "serve with `@-./sides/steamed rice{}`" without pulling the rice's ingredients onto the shopping list of someone who only wants the curry.

### Shopping lists and pantry

Hidden ingredients never reach a shopping list. The [shopping list format](0016-shopping-list-format.md) is unchanged: because the file records recipe references and expands them when read, a reader that skips hidden occurrences produces the right list, and nothing about hidden items needs to be written down. Hidden ingredients are also not offered as optional selections (proposal 0018), even when also marked optional.

Features that match a recipe against available ingredients ("what can I cook?") ignore hidden ingredients entirely: a hidden ingredient is neither required nor counted as missing.

### What hidden does not mean

Hidden is about listing only. It does not say the ingredient is not consumed, or does not count towards nutrition or cost: the salt in "salt to taste" is eaten, the water the pasta boils in mostly is not, and frying oil is somewhere in between. Tools that need that distinction must derive it from elsewhere (for example from the step's verb, or a per-ingredient database) and must not read it off the hidden flag.

### Scope

This proposal standardises the `-` marker and its combination with `?` only. The remaining cooklang-rs component modifiers (`&` reference, `+` new, `@` recipe) stay extensions. They keep working in parsers that enable them, including in combination with `-`, but that behaviour is outside the specification. In particular, cooklang-rs lets a `&` reference inherit the hidden flag from the occurrence it refers to; that is extension behaviour and is not defined here.

### Backward compatibility

Under the current specification `@-water` is a required ingredient named `-water` in practice (strictly, the EBNF excludes punctuation from the first word of a name, so it should not parse as an ingredient at all; cooklang-rs and so CookCLI parse it as `-water`). After this change it is a hidden ingredient named `water`. Ingredient names that intentionally start with `-` are not known to exist, while recipes already using `@-` with the intended hidden meaning do exist, so this change fixes recipes and breaks none in practice. No migration is needed.

Under proposal 0018 alone, `@?-chives` is an optional ingredient named `-chives`; after this change it is an optional hidden ingredient named `chives`. The same argument applies.

## Effect on applications which use Cooklang

### Specification updates

Once accepted, the `-` marker is documented in the language specification (README and EBNF, extending the "Optional ingredients and cookware" section into a section on markers) with canonical tests, the canonical tests README gains the `hidden` property, and the shopping list and pantry sections of the conventions note that hidden components are skipped.

### Display

1. **Inline in steps.** Hidden components render exactly like listed ones: highlighted, with quantity, scaled. Applications may add a subtle marker in editing contexts (for example a dimmer highlight in the editor) so the author can see the flag, but should not clutter cooking mode with it.
2. **Ingredient and cookware lists.** Hidden entries are left out. Applications should not show a "hidden" group by default; an editor or debugging view may offer one.

### Shopping lists

Hidden ingredients are left out when a recipe is added to a shopping list and are not offered as optional extras. No flag to include them is needed: an author who wanted an item on the list would not have hidden it.

### Pantry matching

Hidden ingredients are ignored when matching a recipe against a pantry, in both directions: they are not required and not reported as missing.

### CookCLI (terminal and web-server)

CookCLI should:

1. Parse the `-` marker unconditionally, without enabling the rest of the component modifiers extension (today it parses with all extensions disabled).
2. Leave hidden ingredients and cookware out of the ingredient and cookware lists in all recipe outputs: human, markdown and the web UI (recipe view and cooking mode), while still rendering them inside steps. Expose a `hidden` field in JSON and YAML output.
3. In `shopping-list`, skip hidden ingredients.
4. In pantry-based recipe matching (the `pantry` sub-command and the web UI), ignore hidden ingredients.
5. In `doctor` and the `cooklang-format` crate, keep the marker when reformatting, and warn on a duplicated marker.

No new sub-command is required.

### Mobile applications

Mobile apps should:

1. Leave hidden items out of the recipe's ingredient and cookware tabs, while rendering them highlighted inside steps and in cooking mode.
2. Skip hidden ingredients when adding a recipe to the shopping list, and when matching against the pantry.

### Editors

The VS Code extension, the desktop editor and syntax highlighters should colour the `-` marker as part of the sigil, together with `?`, so `@-water` reads as a marked ingredient rather than a typo. The AI editor and importers may emit `@-` for water, greasing fat and "to taste" seasonings when converting recipes.

### Parsers

Parsers that implement this as an extension today (cooklang-rs) should make the `-` marker part of the core syntax, available with all extensions disabled, following the pattern used for `?`. Other parsers (cooklang-ts, cooklang-swift, cooklang-c, etc.) need the grammar change and the new `hidden` property.

## Alternatives considered

### Write hidden items as plain text

This is what most authors do today. It needs no syntax, but the item loses highlighting, scaling, preparation notes and visibility to any tool that reads steps. A stock recipe that cannot scale its water is a worse recipe. The marker exists to keep the markup while fixing the lists.

### Hide by configuration instead of in the recipe

A per-user "never list these" set (in the pantry or aisle configuration) could drop water and salt from every shopping list. That solves a different problem. Whether an ingredient is a thing to buy is a property of the recipe, decided by its author, and varies by occurrence: the water in bread dough is tap water, but the water in a brine recipe may be most of what the recipe is about, and the salt rubbed on a joint should be bought while a pinch in boiling water should not. Per-user configuration also does not travel with the recipe. The two mechanisms are complementary, and nothing here prevents a pantry convention from suppressing further items.

### A metadata list of hidden ingredients

```yaml
hidden: [water, salt]
```

This hides by name, so it cannot distinguish the salt that is bought from the salt that is not, breaks when the name is spelled differently in different steps, and separates the intent from the place where the author writes the ingredient. It also does not extend naturally to cookware or recipe references.

### Marker inside the braces: `@water{3%l%hidden}` or `@water{-3%l}`

This overloads the quantity slot, conflicts with free-text quantities and negative-looking text, and forces braces onto single-word ingredients. It is also incompatible with the recipes that already use the prefix form.

### Suffix marker or prefix before the sigil: `@water{}-`, `-@water{}`

Both appear in the wild (`-@Cold water{to cover}` in discussion #127). A trailing `-` is ambiguous with dashes in prose and with the `--` comment syntax. A leading `-` before `@` is ambiguous with list-like prose ("- @flour, - @sugar") and with the `-` that starts a comment or a note in surrounding Markdown. Putting the marker between the sigil and the name, as `?` already does, has no such ambiguity and matches the deployed syntax.

### A different character

`!`, `_` or `~` were considered. `_` is a word character and would silently swallow names that start with an underscore; `~` is the timer sigil; `!` has no deployed base. `-` reads as "minus this from the list", is already in use, and is already specified in cooklang-rs.

### Make hidden mean "not consumed"

Nutrition and cost tools would like to know which ingredients end up on the plate. Hidden is not that flag: "salt to taste" is hidden and eaten, pasta water is hidden and poured away, and marinade is listed and mostly discarded. Overloading `-` would make the shopping list and the nutrition count fight over the same bit. Consumption is left to a future proposal.

### Standardise all component modifiers at once

As in proposal 0018, promoting the whole cooklang-rs modifiers extension was rejected because the modifiers are independent features with very different levels of maturity. `-` has simple semantics, is deployed, and causes visible inconsistencies between tools today, so it is taken on its own.

## Acknowledgments

Thanks to [Zheoni](https://github.com/Zheoni) for designing and implementing the modifiers extension in cooklang-chef and cooklang-rs, where this syntax originated, and to the authors of discussion [#127](https://github.com/cooklang/spec/discussions/127) and [cooklang-rs#66](https://github.com/cooklang/cooklang-rs/issues/66) for showing the need in the wild.
