# Produces metadata for unit-based scaling

* Proposal: [0017-produces-metadata](0017-produces-metadata.md)
* Authors: [Alexey Dubovskoy](https://github.com/dubadub)
* Status: **Draft**

## Introduction

This proposal introduces a new canonical metadata key `produces` (with synonyms `output` and `makes`) to declare what a recipe produces in measurable units. This resolves a conflict where `yield` serves as both a synonym for `servings` and the key for unit-based scaling.

Discussion thread: TBD

## Motivation

The canonical metadata table (proposal 0007) lists `yield` as a synonym for `servings`/`serves`, meaning "how many people the recipe feeds." However, the experimental unit-based scaling feature (documented in conventions) also uses `yield` to declare measurable output, e.g. `yield: 500%ml`.

These are fundamentally different concepts:

- **Servings** answers "how many people does this feed?" — e.g. `servings: 4`
- **Produces** answers "how much does this make?" — e.g. `produces: 500%ml`

A sauce recipe might feed 4 people *and* produce 500ml. With `yield` overloaded for both meanings, a recipe can't express both, and parsers can't reliably distinguish intent. Separating these into distinct keys resolves the ambiguity.

## Proposed solution

Add a new canonical metadata key group for declaring recipe output in measurable units:

| Key | Purpose | Example value |
| --- | --- | --- |
| `produces`, `output`, `makes` | What the recipe produces in measurable units. Used for unit-based scaling of referenced recipes. Supports a single value or a list. | `500%ml`, `[500%ml, 350%g]` |

The existing `servings`/`serves`/`yield` row remains unchanged — `yield` stays a valid synonym for servings.

### Single output

A recipe declares what it produces:

```cooklang
---
produces: 500%ml
---

Melt @butter{100%g} in a #saucepan{} over low heat.
Whisk in @flour{50%g} and cook for ~{2%minutes}.
Gradually add @milk{500%ml}, stirring constantly.
```

When referenced as `@./sauces/bechamel{200%ml}`, the scaling factor is `200 / 500 = 0.4`.

### Multiple outputs

A recipe can declare multiple output measurements. This is useful when the same recipe can be referenced by different units:

```cooklang
---
produces:
  - 500%ml
  - 350%g
---

Melt @butter{100%g} in a #saucepan{} over low heat.
Whisk in @flour{50%g} and cook for ~{2%minutes}.
Gradually add @milk{500%ml}, stirring constantly.
```

- `@./sauces/bechamel{200%ml}` matches the `500%ml` entry, scaling factor `0.4`
- `@./sauces/bechamel{175%g}` matches the `350%g` entry, scaling factor `0.5`

When a reference's unit doesn't match any entry in `produces`, it is an error.

### Interaction with servings

`produces` and `servings` are independent and can coexist:

```cooklang
---
servings: 4
produces: 500%ml
---
```

- `@./sauces/bechamel{6%servings}` scales by servings (factor `1.5`)
- `@./sauces/bechamel{200%ml}` scales by output (factor `0.4`)
- `@./sauces/bechamel{2}` scales by the plain numeric factor (`2`)

## Detailed design

### Metadata key syntax

The `produces` key follows standard YAML front matter syntax. Synonyms `output` and `makes` are treated identically.

Single value:
```yaml
---
produces: 500%ml
---
```

List value:
```yaml
---
produces:
  - 500%ml
  - 350%g
---
```

The value format is `number%unit`, consistent with how quantities are expressed elsewhere in Cooklang (ingredient amounts, pantry configuration, shopping lists).

### Scaling resolution

When a recipe reference includes a unit (e.g., `@./recipe{150%ml}`):

1. Look up the referenced recipe's `produces` metadata (or its synonyms `output`/`makes`)
2. Find the entry whose unit matches the reference's unit (case-insensitive)
3. Calculate the scaling factor: `requested_quantity / produces_quantity`
4. Apply the factor to all scalable ingredients

If `produces` is absent or no unit matches, it is an error.

### Precedence

When a reference has a unit that could match either `produces` or `servings` (e.g. a contrived `servings: 4%ml`), `produces` takes precedence for non-`servings` units. The `servings` pseudo-unit is always resolved against the `servings`/`serves`/`yield` key.

## Effect on applications which use Cooklang

### Conventions updates

1. Add `produces`/`output`/`makes` to the canonical metadata table
2. Update the "Scaling Referenced Recipes" section (item 3, Units) to reference `produces` instead of `yield`

### CookCLI (terminal and web-server)

CookCLI should:
1. Recognise `produces`/`output`/`makes` metadata keys
2. Display recipe output information in recipe read output
3. Use `produces` for unit-based scaling in recipe read and shopping list generation

### Mobile applications

Mobile apps should:
1. Display `produces` metadata in recipe detail views
2. Use `produces` for unit-based scaling calculations

## Alternatives considered

### Move unit-based output out of metadata entirely

Instead of a metadata key, use a special syntax line in the recipe body (e.g., `= yield: 500%ml`). This was rejected because the information is declarative metadata about the recipe, not a step instruction. YAML front matter is the right place for it.

### Deprecate or remove `yield` as a servings synonym

This would break existing recipes that use `yield: 4` to mean "serves 4 people." Since `yield` is a common term in recipe formats, keeping it as a servings synonym avoids unnecessary migration and remains intuitive. A future proposal can revisit this if the community prefers.

### Repurpose `yield` exclusively for unit-based output

This was considered but rejected because it would be a breaking change for recipes already using `yield` to mean servings, and the word "yield" is ambiguous on its own — it could mean either concept.

## Acknowledgments

Thanks to the Cooklang community for feedback on unit-based scaling and the discussions that identified this conflict.
