Table of Contents
=================

* [About Cooklang](#about-cooklang)
* [The .cook Recipe Specification](#the-cook-recipe-specification)
   * [Ingredients](#ingredients)
   * [Steps](#steps)
   * [Comments](#comments)
   * [Metadata](#metadata)
   * [Cookware](#cookware)
   * [Timer](#timer)
* [Shopping Lists](#shopping-lists)
* [Conventions](#conventions)
   * [Adding Pictures](#adding-pictures)
   * [Canonical metadata](#canonical-metadata)
* [Advanced](#advanced)
   * [Notes](#notes)
   * [Sections](#sections)
   * [Short-hand preparations](#short-hand-preparations)
* [Projects Which Use Cooklang](#projects-which-use-cooklang)
* [Syntax Highlighting](#syntax-highlighting)

## About Cooklang
Cooklang is the markup language at the center of an open-source ecosystem for cooking and recipe management. In Cooklang, each text file is a recipe written as plain-english instructions with markup syntax to add machine-parsible information about required ingredients, cookware, time, and metadata.

## The .cook recipe specification
Below is the specification for defining a recipe in Cooklang.

### Ingredients

To define an ingredient, use the `@` symbol. If the ingredient's name contains multiple words, indicate the end of the name with `{}`.

```
Then add @salt and @ground black pepper{} to taste.
```

To indicate the quantity of an item, place the quantity inside `{}` after the name.

```
Poke holes in @potato{2}.
```

To use a unit of an item, such as weight or volume, add a `%` between the quantity and unit.

```
Place @bacon strips{1%kg} on a baking sheet and glaze with @syrup{1/2%tbsp}.
```

### Steps

Each paragraph in your recipe file is a cooking step. Separate steps with an empty line.

```
A step,
the same step.

A different step.
```

### Comments
You can add comments up to the end of the line to Cooklang text with `--`.
```
-- Don't burn the roux!

Mash @potato{2%kg} until smooth -- alternatively, boil 'em first, then mash 'em, then stick 'em in a stew.
```

Or block comments with `[- comment text -]`.
```
Slowly add @milk{4%cup} [- TODO change units to litres -], keep mixing
```

### Metadata
Recipes are more than just steps and ingredients—they also include context, such as preparation times, authorship, and dietary relevance. You can add metadata to your recipe using YAML front matter, add `---` at the beginning of a file and `---` at the end of the front matter block.

```yaml
---
title: Spaghetti Carbonara
tags:
  - pasta
  - quick
  - comfort food
---
```

### Cookware
You can define any necessary cookware with `#`. Like ingredients, you don't need to use braces if it's a single word.
```
Place the potatoes into a #pot.
Mash the potatoes with a #potato masher{}.
```

### Timer
You can define a timer using `~`.
```
Lay the potatoes on a #baking sheet{} and place into the #oven{}. Bake for ~{25%minutes}.
```

Timers can have a name too:

```
Boil @eggs{2} for ~eggs{3%minutes}.
```

Applications can use this name in notifications.

## Shopping Lists
To support the creation of shopping lists by apps and the command line tool, Cooklang includes a specification for a configuration file to define how ingredients should be grouped on the final shopping list.
You can use `[]` to define a category name. These names are arbitrary, so you can customize them to meet your needs. For example, each category could be an aisle or section of the store, such as `[produce]` and `[deli]`.
```
[produce]
potatoes

[dairy]
milk
butter
```
Or, you might be going to multiple stores, in which case you might use `[Tesco]` and `[Costco]`.
```
[Costco]
potatoes
milk
butter

[Tesco]
bread
salt
```
You can also define synonyms with `|`.
```
[produce]
potatoes

[dairy]
milk
butter

[deli]
chicken

[canned goods]
tuna|chicken of the sea
```

## Conventions

There're things which aren't part of the language specification but rather common conventions used in tools build on top of the language.

### Adding Pictures
You can add images to your recipe by including a supported image file (`.png`,`.jpg`) matching the name of the recipe recipe in the same directory.
```
Baked Potato.cook
Baked Potato.jpg
```
You can also add images for specific steps by including a step number before the file extension.
```
Chicken French.cook
Chicken French.0.jpg
Chicken French.3.jpg
```

### Canonical metadata

To use your recipes across different apps, follow the conventions on how to name metadata in common cases:

| Key | Purpose | Example value |
| --- | --- | --- |
| `source`, `source.name` | Where the recipe came from. Usually a URL, can also be text (eg. a book title). | `https://example.org/recipe`, `The Palomar Cookbook <urn:isbn:9781784720995>`, `mums` |
| `author`, `source.author` | The author of the recipe. | `John Doe` |
| `source.url`|The URL of the recipe if nested format is used.|`https://example.org/recipe`|
| `servings`, `serves`, `yield` | Indicates how many people the recipe is for. Used for scaling quantities. Leading number is used for scaling, anything else is ignored but shown as units. | `2`,`15 cups worth` |
| `course`, `category` | Meal category or course. | `dinner` |
| `locale` | The locale of the recipe. Used for spelling/grammar during edits, and for pluralisation of amounts. Uses ISO 639 language code, then optionally an underscore and the ISO 3166 alpha2 "country code" for dialect variants | `es_VE`, `en_GB`, `fr`  |
| `time required`, `time` or `duration` | The preparation + cook time of the recipe. Various formats can be parsed, if in doubt use `HhMm` format to avoid plurals and locales. | `45 minutes`, `1 hour 30 minutes`,`1h30m` |
| `prep time`, `time.prep`|Time for preparation steps only.|`2 hour 30 min`|
| `cook time`, `time.cook`|Time for actual cooking steps.|`10 minutes`|
| `difficulty`|Recipe difficulty level.|`easy`|
| `cuisine`|The cuisine of the recipe.|`French`|
| `diet`|Indicates a dietary restriction or guideline for which this recipe or menu item is suitable, e.g. diabetic, halal etc.|`gluten-free`, or array of values|
| `tags`|List of descriptive tags.|`[2022, baking, summer]`|
| `image`, `images`, `picture`, `pictures`|URL to a recipe image.|`https://example.org/recipe_image.jpg` or array of URLs|
| `title`|Title of the recipe.|`Uzbek Manti`|
| `introduction`, `description`|Additional notes about the recipe.|`This recipe is a traditional Uzbek dish that is made with a variety of vegetables and meat.`|

## Advanced

### Notes
To include relevant background, insights, or personal anecdotes that aren't part of the cooking steps, use notes. Start a new line with `>` and add your story.

```
> Don't burn the roux!

Mash @potato{2%kg} until smooth -- alternatively, boil 'em first, then mash 'em, then stick 'em in a stew.
```

### Sections
Some recipes are more complex than others and may include components that need to be prepared separately. In such cases, you can use the section syntax, e.g., `==Dough==`. The section name and the `=` symbols after it are optional, and the number of `=` symbols does not matter.

```cooklang
= Dough

Mix @flour{200%g} and @water{100%ml} together until smooth.

== Filling ==

Combine @cheese{100%g} and @spinach{50%g}, then season to taste.
```

### Short-hand preparations

Many recipes involve repetitive ingredient preparations, such as peeling or chopping. To simplify this, you can define these common preparations directly within the ingredient reference using shorthand syntax:

```cooklang
Mix @onion{1}(peeled and finely chopped) and @garlic{2%cloves}(peeled and minced) into paste.
```

These preparations will be clearly displayed in the ingredient list, allowing you to get everything ready before you start cooking.

## Projects Which Use Cooklang

* [Cooklang playground](https://biowaffeln.github.io/cooklang-parser/)
* [Obsidian plugin](https://github.com/deathau/cooklang-obsidian)
* [Official command line application](https://github.com/cooklang/CookCLI)
* [Official iOS application](https://cooklang.org/app/)


## Syntax Highlighting

* [Emacs](https://github.com/cooklang/cook-mode)
* [Nano](https://github.com/le-jun/cooklang.nanorc)
* [SublimeText](https://github.com/cooklang/CookSublime)
* [Vim](https://github.com/luizribeiro/vim-cooklang)
* [VSCode](https://github.com/cooklang/CookVSCode)
