Table of Contents
=================

* [About Cooklang](#about-cooklang)
* [The .cook Recipe Specification](#the-cook-recipe-specification)
   * [Ingredients](#ingredients)
   * [Comments](#comments)
   * [Metadata](#metadata)
   * [Cookware](#cookware)
   * [Timer](#timer)
* [The Shopping List Specification](#the-shopping-list-specification)
* [Conventions](#conventions)
   * [Adding Pictures](#adding-pictures)
* [Roadmap](#roadmap)
* [Parser Imlementation](#parser-implementation)
* [Syntax Highlighting](#syntax-highlighting)

## About Cooklang
Cooklang is the markup language at the center of an open-source ecosystem for cooking and recipe management. In Cooklang, each text file is a recipe written as plain-english instructions with markup syntax to add machine-parsible information about required ingredients, cookware, time, and metadata. 

## The .cook Recipe Specification
Below is the specification for defining a recipe in cooklang. 

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

### Comments
You can add comments to CookLang text with `//`.
```
// Don't burn the roux!

Mash @potato{2%kg} until smooth // alternatively, boil 'em first, then mash 'em, then stick 'em in a stew.
Slowly add @milk{4%cup}, keep mixing
```

### Metadata
You can add metadata tags to your recipe for information such as source (or author), meal, total prep time, and number of people served.
```
>> source: https://www.gimmesomeoven.com/baked-potato/
>> time required: 1.5 hours
>> course: dinner
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

## The Shopping List Specification
To support the creation of shopping lists by apps and the command line tool, CookLang includes a specification for a configuration file to define how ingredients should be grouped on the final shopping list.
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

## Roadmap

### Servings (proposal)
You can manually add information for scaling serving size up or down. Serving size information comes in two parts: the metadata tag, and ingredient tags.
The metadata tag defines the serving sizes the recipe supports.

```
>> servings: 2|4|8
```

Then, you can automatically scale ingredient quantities with `*`. This will multiply the quantity given by the number of servings selected.

```
>> servings: 2|4|8
Add @milk{1/2*%cup} and mix until smooth.
```
Alternatively, you can manually specify ingredient quantities for each serving size. This is useful for non-linear scaling.

```
>> servings: 2|4|8
Add @milk{1|2|3%cup} and mix until smooth.
```

If no ingredient scaling is defined, the same quantity will be used for all serving sizes.

```
>> servings: 2|4|8
Add @salt{1%tsp} // this is the same
Add @salt{1|1|1%tsp} // as this
```

## Parser Implementation

* [Swift](https://github.com/cooklang/CookInSwift)
* [Rust](https://github.com/umgefahren/cook-with-rust) WIP

## Syntax Highlighting

* [Emacs](https://github.com/cooklang/cook-mode)
* [SublimeText](https://github.com/cooklang/CookSublime)
* [Vim](https://github.com/luizribeiro/vim-cooklang)
* [VSCode](https://github.com/cooklang/CookVSCode)
