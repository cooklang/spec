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
* [Conventions](conventions.md) (shopping lists, pantry, scaling, pictures, canonical metadata)
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

If you want to force a line break within a step, end the line with a backslash `\`. The backslash will be replaced with a newline character when rendered.

```cooklang
Lay out the @rice paper{1}.\
Top with @avocado{1/2}(sliced),\
@cucumber{1/2}(julienned),\
and @cooked shrimp{4}.
```

This will render as multiple lines within a single step.

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

You can reference other recipes using the existing `@` ingredient syntax. The path is relative to the root of the recipes directory, without the `.cook` extension:

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
