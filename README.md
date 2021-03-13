### Comments
```
// this is a commentary

Prepare bla-bla-bla // this will be ommited 
add some soup.
```

### Ingrediens

In the easiest case just put  `@` symbol before ingredient to mark it:
```
Add @salt and @pepper
````

In some cases ingredients can have multiple words. Add `{}` to show where is the end:
```
Add @garlic salt{} and @black pepper{} 
```

Most of the time you will need to add amount of ingredients:
```
@apple{3} vs default to items

```

If you need to set units add them with `%`:
```
Add @chiken{1.5%kg} and @salt{pinch}
```

~~If you need to add some modifier add that in parenthesis:~~
```
Add @onion{3%medium}(finely chopped) and fry in @olive oil{2%tbsp}
```

It understands fractions too:

```
Add @vanilla syrope{1/4%tsp}
```

### Metadata

In any place of your recipe you can add metadata in `key: value` fashion:

```
>> course: breakfast
>> time required: 3 hours
```



### Servings
To add servings first add metadata listing how many servings the recipe accomodates
```
>> servings: 2|4|5
```

Then in you description specify amount for each serving:

```
Add @potatoes{3|5|7%medium} 
```

Or to specify proportionally use `*`:

```
Add @potatoes{3*%medium}

// is the same as

Add @potatoes{3|6|7.5%medium}

```

If nothing specified the same value will be used for all servings:

```
Add @salt{1%tsp}

// is the same as

Add @salt{1|1|1%tsp}

```

First serving is default one and will be used when adding in bulk

### Nutrition
per serving? total?
```
>> nutrition.energy: 550 kcal
>> nutrition.carbonates: 12
```


### Equipment

If you like to link equipment used use `#`

```
Fry on a #pan or #ironed thing{}
```


### Timer

```
Put everything in a dish and place into oven for ~{40%m}.
```

TODO need to define conversion with config?

### Aisle config file

Use this format. Parser understands plural and singular and merge. You can separate synonyms with `|` (TODO)
```
 
[fruit and veg]
carrots
celery
dill
garlic
beetroots
potatoes
mushrooms
onion
bell pepper


[milk and dairy]
butter

  
[meat and seafood]
chicken

[breads and baked goods]  

[tinned goods and baking]
cannelini beans

[packaged goods, pasta and sauces]
tomato paste

[dried herbs and spices]
bay leaves
black pepper|ground pepper
cayenne pepper
salt|sea salt

[oils and dressings]
olive oil
white vinegar
white wine
```


### Othere locale support
TODO

### Pics

To add a recipe picture add image file (png, jpg) alongside with the same name as your cook file.

```
Chicken French.cook
Chicken French.jpg
```

To add a step picture add image file with the same name as you cook file with index of step starting from 0:

```
Chicken French.cook
Chicken French.0.jpg
Chicken French.3.jpg
```

If not added default patterns will be shown.

### Nutrition values

You can specify nutrition values for you dish. This way when creating a meal plan cook can show balance


### Best practices

You can follow [[Cook/Best Practices]] to make less efortless.
