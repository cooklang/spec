## Roadmap

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

Alternatively, you can define the same quantity for all serving sizes with `!`.

```
>> servings: 2|4|8
Add @salt{1!%tsp} -- this is the same
Add @salt{1|1|1%tsp} -- as this
```
