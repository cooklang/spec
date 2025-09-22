> Just heads up, that not all latest language features supported in the apps yet.
> You can track progress at https://github.com/orgs/cooklang/projects/4

* [About Cooklang](#about-cooklang)
* [The .cook Recipe Specification](#the-cook-recipe-specification)
   * [Ingredients](#ingredients)
   * [Steps](#steps)
   * [Comments](#comments)
   * [Metadata](#metadata)
   * [Cookware](#cookware)
   * [Timer](#timer)
* [Shopping Lists](#shopping-lists)
* [Pantry Configuration](#pantry-configuration)
* [Scaling and Servings](#scaling-and-servings)
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

```cooklang
Then add @salt and @ground black pepper{} to taste.
```

To indicate the quantity of an item, place the quantity inside `{}` after the name.

```cooklang
Poke holes in @potato{2}.
```

To use a unit of an item, such as weight or volume, add a `%` between the quantity and unit.

```cooklang
Place @bacon strips{1%kg} on a baking sheet and glaze with @syrup{1/2%tbsp}.
```

Now you can try Cooklang and experiment with a few things in the [Cooklang Playground](https://cooklang.github.io/cooklang-rs/)!

### Steps

Each paragraph in your recipe file is a cooking step. Separate steps with an empty line.

```cooklang
A step,
the same step.

A different step.
```

### Comments
You can add comments up to the end of the line to Cooklang text with `--`.
```cooklang
-- Don't burn the roux!

Mash @potato{2%kg} until smooth -- alternatively, boil 'em first, then mash 'em, then stick 'em in a stew.
```

Or block comments with `[- comment text -]`.
```cooklang
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
```cooklang
Place the potatoes into a #pot.
Mash the potatoes with a #potato masher{}.
```

### Timer
You can define a timer using `~`.
```cooklang
Lay the potatoes on a #baking sheet{} and place into the #oven{}. Bake for ~{25%minutes}.
```

Timers can have a name too:

```cooklang
Boil @eggs{2} for ~eggs{3%minutes}.
```

Applications can use this name in notifications.

## Shopping Lists
To support the creation of shopping lists by apps and the command line tool, Cooklang includes a specification for a configuration file to define how ingredients should be grouped on the final shopping list.
You can use `[]` to define a category name. These names are arbitrary, so you can customize them to meet your needs. For example, each category could be an aisle or section of the store, such as `[produce]` and `[deli]`.
```toml
[produce]
potatoes

[dairy]
milk
butter
```
Or, you might be going to multiple stores, in which case you might use `[Tesco]` and `[Costco]`.
```toml
[Costco]
potatoes
milk
butter

[Tesco]
bread
salt
```
You can also define synonyms with `|`.
```toml
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

There are things which aren't part of the language specification but rather common conventions used in tools built on top of the language.

### Adding Pictures
You can add images to your recipe by including a supported image file (`.png`,`.jpg`) matching the name of the recipe in the same directory.
```sh
Baked Potato.cook
Baked Potato.jpg
```
You can also add images for specific steps by including a step number before the file extension.
```sh
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

## Pantry Configuration

Cooklang supports a pantry inventory file in TOML format to track ingredients you have on hand. This file helps with meal planning and shopping list generation.

### Format

The pantry file uses TOML sections to organize items by storage location:

```toml
[freezer]
cranberries = "500%g"
spinach = { bought = "05.05.2024", expire = "05.06.2025", quantity = "1%kg" }

[fridge]
milk = { expire = "10.05.2024", quantity = "1%L" }

[pantry]
rice = "5%kg"
```

### Supported Attributes

Each item can be specified as either:
- A simple quantity string: `"500%g"`
- An object with attributes:
  - `bought`: Date when the item was purchased (e.g., "05.05.2024")
  - `expire`: Expiration date of the item (e.g., "05.06.2025")
  - `quantity`: Amount using Cooklang quantity format (e.g., "1%kg")
  - `low`: Low stock threshold for alerts (e.g., "100%g")

### Example

```toml
[freezer]
ice_cream = "2%L"
frozen_peas = { bought = "01.01.2024", quantity = "500%g", low = "200%g" }

[fridge]
cheese = { expire = "15.05.2024" }
yogurt = { bought = "05.05.2024", expire = "12.05.2024", quantity = "500%ml" }

[pantry]
flour = "5%kg"
pasta = { quantity = "1%kg", low = "200%g" }
```

Applications can use this data to check ingredient availability, track expiration dates, and generate shopping lists based on what's running low.

## Scaling and Servings

Cooklang supports automatic recipe scaling based on servings. This allows users to adjust recipes for different numbers of people.

### Defining Servings

Specify the default serving size in the metadata:

```yaml
---
servings: 2
---
```

If not specified, recipes default to 1 serving. All ingredient quantities in the recipe are written for this default serving size.

### Scaling Behavior

#### Linear Scaling (Default)
Most ingredients scale linearly with servings:

```cooklang
Add @milk{1/2%cup} and mix until smooth.
```

When scaling from 2 to 4 servings, the milk quantity doubles to 1 cup.

#### Fixed Quantities
Some ingredients shouldn't scale. Use `=` to lock the quantity:

```cooklang
Season with @salt{=1%tsp} to taste.
```

This keeps salt at 1 tsp regardless of serving size.

### What Doesn't Scale

- **Timers**: Cooking times typically remain the same regardless of portion size
- **Cookware**: Pan and pot sizes don't automatically adjust

### Example

```cooklang
---
servings: 4
---

Mix @flour{500%g} with @water{300%ml}.
Add @yeast{=1%packet} and let rise for ~{1%hour}.
```

When scaled to 8 servings:
- Flour becomes 1000g (doubled)
- Water becomes 600ml (doubled)
- Yeast stays at 1 packet (fixed)
- Rising time remains 1 hour (timers don't scale)

## Advanced

### Notes
To include relevant background, insights, or personal anecdotes that aren't part of the cooking steps, use notes. Start a new line with `>` and add your story.

```cooklang
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

### Referencing other recipes

You can reference other recipes using the existing `@` ingredient syntax, inferring relative file paths from the ingredient name:

```cooklang
Pour over with @./sauces/Hollandaise{150%g}.
```

These preparations should be clearly displayed in the ingredient list, allowing you to get everything ready before you start cooking.

## Projects Which Use Cooklang
* [Cooklang playground](https://cooklang.github.io/cooklang-rs/)
* [Obsidian plugin](https://github.com/deathau/cooklang-obsidian)
* [Official command line application](https://github.com/cooklang/CookCLI)
* [Community alternative command line application](https://github.com/Zheoni/cooklang-chef)
* [Official iOS application](https://cooklang.org/app/)
* [Official Android application](https://cooklang.org/app/)


## Syntax Highlighting

* [Emacs](https://github.com/cooklang/cook-mode)
* [Nano](https://github.com/le-jun/cooklang.nanorc)
* [SublimeText](https://github.com/cooklang/CookSublime)
* [Vim](https://github.com/luizribeiro/vim-cooklang)
* [VSCode](https://github.com/cooklang/CookVSCode)
* More options: See [syntax highlighting documentation](https://cooklang.org/docs/syntax-highlighting/).

## Roadmap

There's a GitHub board where we show what we're working on and what's next https://github.com/orgs/cooklang/projects/4.
