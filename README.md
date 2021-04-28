Table of Contents
=================

* [About Cooklang](#about-cooklang)
* [The .cook Recipe Specification](#the-cook-recipe-specification)
   * [Ingredients](#ingredients)
   * [Comments](#comments)
   * [Metadata](#metadata)
      * [Servings](#servings)
   * [Cookware](#cookware)
   * [Timer](#timer)
* [Adding Pictures](#adding-pictures)
* [The Shopping List Specification](#the-shopping-list-specification)

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

To modify the ingredient any other way, use parentheses.
```
Top with @green onions{1%tbsp}(finely chopped)
```

### Comments
You can add comments to cooklang text with `//`. 
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

#### Servings
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

## Adding Pictures
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

## The Shopping List Specification
The shopping list parser can use a configuration file and group items for a shopping list by category.
The parser understands both plural and singular forms. You can separate synonyms with `|`.
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
tuna|chicken of the sea

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
