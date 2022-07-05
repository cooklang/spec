## Roadmap

### Internationalization and localization

#### Locale lookup

App should guess locale from the system.

If config has locale set it should use one instead. In file `./config/main.conf` add line `locale = ru_RU`. Or simple `locale = ru`.

Locale of recipe can be set in file name or metadata. Name file `Koldskål.da.cook`. Or in metadata set

```
>> locale: da_DK
```

#### Inflection rules for locale

- how to singularise ingredients and convert case to base when parsing
- how to singularise units and convert case to base when parsing
- how to pluralise units and ingredients when display with numbers (this can be done by Apple?)

For many languages plural and singular form of an ingredient or unit will be different. To keep recipe readable

https://unicode-org.github.io/cldr-staging/charts/latest/supplemental/language_plural_rules.html
https://cldr.unicode.org/translation/grammatical-inflection
Depending on the locale, there could be up to 6 plural forms used: zero, one, two, few, many, other.



### Servings

You can manually add information for scaling serving size up or down. Serving size information comes in two parts: the metadata tag, and ingredient tags.

The metadata tag defines the serving sizes the recipe supports.

```
>> servings: 2|4|8
```

You can manually specify ingredient quantities for each serving size. This is useful for non-linear scaling.

```
>> servings: 2|4|8
Add @milk{1|2|3%cup} and mix until smooth.
```

If no ingredient scaling is defined, this will multiply the quantity given by the number of servings selected.

```
>> servings: 2|4|8
Add @salt{1%tsp} -- this is the same
Add @salt{1|2|4%tsp} -- as this
```

Mentioned serving options will be available in UI. But it's possible to scale to a number of servings which is not defined here. The parser will inter- or extrapolate if necessary.

Default value for servings will be the first one. It's not required to go always up or down.

If servings not specified, it considered to be 1.

#### TODO

* How to scale cookware, timer?
* How it works with include

### Optional ingredients

Use `?` at the end of an ingredient name to indicate that an ingredient is not required:

```
Add some @?rum and @?egg yolk{1}, if desired.
```

#### TODO

* Can ingredients contain a question mark? Will it mess it up?
* Other syntax idea:

```
Add some @rum? and @egg yolk?{1}, if desired.
```


### Variants

Separate ingredients with '|' to define alternatives.

```
Rinse the @bream{450%g} | @cod{450%g} | @seabass{300%g} | @catfish{450%g}, remove any scales that may remain, as well as fins.
```

It's allowed to have some text in between the ingredients too. Just not forget to place '|' somewhere in between ingredients.

```
For easy-blend @dried yeast{5%g}, stir this into the flour. | For @fresh yeast{10%g}, crumble it and rub into the flour as you would with butter when making pastry.
```

It's possible to make variant optional too. If only one of ingredients optional parser will make them all optional too.

#### TODO

* Pipe's not really visible in text
* Optional is kind of messy, how can they play well together
* how to calculation nutrition, by first ingredient?


### Including other recipes

One can reuse recipes and include them as a file relative to the recipe file. The number shows how much of default servings to use from the recipe.

```
Add @../Sauces/Guacamole.cook{1/10} and mix well.
```

It can be optional

```
Top with @?../Sauces/Enchilada sauce.cook{}.
```

If not specified default is 1.

It can be one of variants:

```
Add @pesto sauce{100%g} | @../Sauces/Pesto sauce.cook{1/3} and mix well.
```

Scaling works the same way, if non-linear scaling required one can specify

```
>> servings: 2|4|8
Add @../Sauces/Guacamole.cook{1/10|1/4|1/2} and mix well.
```

#### TODO
* support recursion?
* detect circular deps
* in UI it's rendered with links

### Canonical metadata

* Recipe description
* Image from metadata
* Custom image lookup
* Tags
* Difficulty
* Nutrition facts
* Source
* Title
* locale

### Ingredients metadata

For each ingredient we can define metadata.

```yaml
# filename potato.yaml
conversion:
  - "50%g = 1%medium"
  - "0.7%kg = 1%l"
nutrition:
 ...
```

### Units conversion

For mass/mass, volume/volume I think to use conversion table in this way (example for people from Europe):
```
1%l = 1000%ml
236.6%ml = 1%cup
1%kg = 1000%g
```
The item on the left is "domestic" and parser will prefer it over the right side when makes the plan for converting units. For the US cups will be on the left. The beauty of this approach is that people tune their level of control, because some prefer to have everything measured in grams and some are fine with approximate values. The problem I can't solve with this approach is how to use different "domestic" units in the same time based on value itself and pick one which is more suitable for formatting, for example parser should prefer `400 ml` over `0.4 l`, but not `1.7 cups` 🤔.

For mass/volume and volume/mass there's no other way, but to introduce an ingredient density: conversion rules which applicable only to one ingredient. I though to have a yaml file per ingredient where we store conversion rules (and later also nutrition values):

```yaml
# filename potato.yaml
conversion:
  - "50%g = 1%medium"
  - "0.7%kg = 1%l"
nutrition:
 ...
```
Perhaps add some sane default values as well which can be used if nothing like that provided. On the left it should always be mass units and on the right volume.

### Nutrition values

### Paragraph

Match Markdown paragraph; empty line separates steps, so it's possible to use line end to format file.

