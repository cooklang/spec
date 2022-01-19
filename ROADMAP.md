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
Add @salt{1%tsp} -- this is the same
Add @salt{1|1|1%tsp} -- as this
```
