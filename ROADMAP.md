## Roadmap

### Internationalization and localization

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
* Source
* Title

### Units conversion

### Nutrition values
