# Conventions

There are things which aren't part of the language specification but rather common conventions used in tools built on top of the language.

## File Types

The Cooklang ecosystem uses the following file types:

- **`.cook` files** — recipes written as plain-text instructions with Cooklang markup.
- **`.menu` files** — meal plans that reference recipes. A `.menu` file is a valid Cooklang file that uses sections for days and recipe references to compose a plan.
- **`.shopping-list`** — a shopping list definition combining recipe references and free-hand ingredients. Hidden file (one per directory).
- **`.shopping-checked`** — an append-only log tracking which ingredients have been checked off. Hidden companion to `.shopping-list`.

### Menu Files

Menu files use sections to organise days (or meals) and recipe references to pull in dishes. Recipe paths are relative to the root recipe directory, without the `.cook` extension.

```cooklang
= Monday

@./mains/pasta carbonara{2}
@./sides/green salad{}

= Tuesday

@./mains/chicken stir fry{4}
@./sides/steamed rice{4}

= Wednesday

@./soups/minestrone{6}
@./breads/focaccia{1}
```

Sections can include dates in `YYYY-MM-DD` format. Applications recognise these dates and can surface a shortcut link to the relevant day's plan (e.g. highlighting today's menu).

```cooklang
== Day 1 (2026-03-07) ==

@./breakfast/shakshuka{4%servings}

== Day 2 (2026-03-08) ==

@./mains/chicken stir fry{4%servings}
```

## Scaling Referenced Recipes

When you reference another recipe with `@./path/to/recipe{quantity}`, the quantity controls how the referenced recipe is scaled:

1. **No units** — scales the whole recipe by the given factor. `@./bread{2}` makes double the recipe.
2. **Servings** — reads the referenced recipe's `servings` metadata and calculates a scaling factor to match. `@./pasta carbonara{4%servings}` for a recipe written for 2 servings will double all quantities.
3. **Units** — reads the referenced recipe's `yield` metadata and calculates a scaling factor based on that. Only matching units are supported. For example, if a sauce recipe has `yield: 500%ml`, then `@./sauces/hollandaise{150%ml}` scales it down to produce 150 ml.

## Adding Pictures
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

Alternatively, you can set an image URL in the recipe metadata using the `image` key (see [Canonical metadata](#canonical-metadata)).

## Canonical metadata

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

## Shopping Lists

### Shopping list format

A shopping list is a pair of hidden files in a directory:

- **`.shopping-list`** — the list definition (recipe references and free-hand ingredients)
- **`.shopping-checked`** — an append-only log of checked/unchecked ingredients

On Unix-like systems these files are hidden by convention (dot-prefix). On Windows, applications should set the hidden file attribute.

#### List definition

The `.shopping-list` file is line-oriented. Recipe references start with `./` and support an optional multiplier in braces. Free-hand ingredients appear at the top level with an optional quantity. Comments use `--` (line/inline) and `[- -]` (block), consistent with Cooklang recipe syntax.

Only recipe references may be nested (indented with 2 spaces per level). Plain ingredients are always top-level. Multiple levels of nesting are supported — a menu can contain recipes, which can contain sub-recipes.

```
./Plans/3 Day Plan I
  ./Breakfast/Mexican Style Burrito{2}
    ./Components/Guacamole{2}
    ./Components/Beans{2}
  ./Salads/Boring{2}
  ./Slowcooker/Slow-cooker beef stew{0.5}
olive oil{4%l}
salt
-- remember to check the pantry
```

#### Check file

The `.shopping-checked` file tracks which ingredients have been acquired. Each line is `+ name` (checked) or `- name` (unchecked). The last entry for a given ingredient wins.

```
+ salt
+ avocados
+ olive oil
- avocados
+ avocados
```

Matching is **case-insensitive** and applies **globally** — `+ butter` checks off butter regardless of which recipe it belongs to.

#### Compaction

Applications can compact the check file by replaying the log, reconciling against the current `.shopping-list` (dropping stale entries), and rewriting with only `+ name` lines for checked ingredients. Compaction is idempotent.

### Ingredient grouping

To support the creation of shopping lists by apps and the command line tool, Cooklang includes a specification for a configuration file to define how ingredients should be grouped on the final shopping list.
You can use `[]` to define a category name. These names are arbitrary, so you can customize them to meet your needs. For example, each category could be an aisle or section of the store, such as `[produce]` and `[deli]`.

The order of sections and ingredients in the configuration file matters — apps display them exactly as written. Arrange sections to match the layout of your store so you can shop aisle by aisle without backtracking.
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
