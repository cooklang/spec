# Yield metadata for unit-based scaling

* Proposal: [0017-yield-metadata](0017-yield-metadata.md)
* Authors: [Alexey Dubovskoy](https://github.com/dubadub)
* Status: **Awaiting review**

## Introduction

This proposal clarifies the canonical metadata keys for recipe output by separating two concepts that have been conflated: how many people a recipe feeds (`servings`/`serves`) and what measurable quantity a recipe produces (`yield`). `yield` is repurposed exclusively for unit-based output, and is removed as a synonym for `servings`.

Discussion thread: [Proposal 0017: yield metadata for unit-based scaling](https://github.com/cooklang/spec/pull/146)

## Motivation

The canonical metadata table (proposal 0007) lists `yield` as a synonym for `servings`/`serves`, meaning "how many people the recipe feeds." However, the experimental unit-based scaling feature (documented in conventions) also uses `yield` to declare measurable output, e.g. `yield: 500%ml`.

These are fundamentally different concepts:

- **Servings** answers "how many people does this feed?" — e.g. `servings: 4`
- **Yield** answers "how much does this make?" — e.g. `yield: 500%ml`

A sauce recipe might feed 4 people *and* produce 500ml. With `yield` overloaded for both meanings, a recipe can't express both, and parsers can't reliably distinguish intent.

The natural English usage aligns with this split: one says "serves 4" or "makes 8 servings" for people, and "yields 500ml" for measurable quantity — matching the [dictionary definition](https://www.merriam-webster.com/dictionary/yield) of yield. Keeping two key groups (`servings`/`serves` for people, `yield` for quantity) is simpler and less ambiguous than introducing a third term.

## Proposed solution

Split the existing `servings`/`serves`/`yield` metadata row into two distinct key groups:

| Key | Purpose | Example value |
| --- | --- | --- |
| `servings`, `serves` | How many people the recipe feeds. | `4` |
| `yield` | What the recipe produces in measurable units. Used for unit-based scaling of referenced recipes. Supports a single value or a list. | `500%ml`, `[500%ml, 350%g]` |

`yield` is removed as a synonym for `servings`. See [Alternatives considered](#alternatives-considered) for the migration rationale.

### Single output

A recipe declares what it yields:

```cooklang
---
yield: 500%ml
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
yield:
  - 500%ml
  - 350%g
---

Melt @butter{100%g} in a #saucepan{} over low heat.
Whisk in @flour{50%g} and cook for ~{2%minutes}.
Gradually add @milk{500%ml}, stirring constantly.
```

- `@./sauces/bechamel{200%ml}` matches the `500%ml` entry, scaling factor `0.4`
- `@./sauces/bechamel{175%g}` matches the `350%g` entry, scaling factor `0.5`

When a reference's unit doesn't match any entry in `yield`, it is an error.

### Interaction with servings

`yield` and `servings` are independent and can coexist:

```cooklang
---
servings: 4
yield: 500%ml
---
```

- `@./sauces/bechamel{6%servings}` scales by servings (factor `1.5`)
- `@./sauces/bechamel{200%ml}` scales by yield (factor `0.4`)
- `@./sauces/bechamel{2}` scales by the plain numeric factor (`2`)

## Detailed design

### Metadata key syntax

The `yield` key follows standard YAML front matter syntax.

Single value:
```yaml
---
yield: 500%ml
---
```

List value:
```yaml
---
yield:
  - 500%ml
  - 350%g
---
```

The value format is `number%unit`, consistent with how quantities are expressed elsewhere in Cooklang (ingredient amounts, pantry configuration, shopping lists).

### Scaling resolution

When a recipe reference includes a unit (e.g., `@./recipe{150%ml}`):

1. Look up the referenced recipe's `yield` metadata
2. Find the entry whose unit matches the reference's unit (case-insensitive)
3. Calculate the scaling factor: `requested_quantity / yield_quantity`
4. Apply the factor to all scalable ingredients

If `yield` is absent or no unit matches, it is an error.

### Migration

`yield` was previously listed as a synonym for `servings`. Any recipe using `yield: 4` (a plain number) to mean "serves 4" should be updated to `servings: 4` or `serves: 4`.

Parsers can distinguish the old and new meanings by value shape:
- `yield: 4` — plain number, old servings usage; should be migrated
- `yield: 500%ml` — `number%unit`, new output usage

Implementations may emit a warning when encountering plain-number `yield` values during a transition period.

## Effect on applications which use Cooklang

### Conventions updates

1. Split the canonical metadata row: `servings`/`serves` (remove `yield`), and add a new row for `yield` as measurable output
2. Keep the "Scaling Referenced Recipes" section (item 3, Units) referencing `yield`, now with a single unambiguous meaning

### CookCLI (terminal and web-server)

CookCLI should:
1. Treat `yield` as measurable output (not a servings synonym)
2. Display yield information in recipe read output
3. Use `yield` for unit-based scaling in recipe read and shopping list generation
4. Optionally warn when a recipe uses `yield` with a plain number value (old usage)

### Mobile applications

Mobile apps should:
1. Display `yield` metadata in recipe detail views alongside `servings`
2. Use `yield` for unit-based scaling calculations

## Alternatives considered

### Introduce a new key (`produces`/`output`/`makes`) and keep `yield` as a servings synonym

An earlier draft of this proposal added `produces` (with synonyms `output` and `makes`) for measurable output and left `yield` as a servings synonym. This was rejected because it expanded the canonical metadata surface with a third concept ("produces") whose meaning overlaps with both existing keys, and left `yield` ambiguous. Reviewers pointed out that natural English already maps cleanly to two groups: "serves/servings" for people and "yield" for quantity produced. Reusing `yield` with a single, precise meaning is simpler.

### Move unit-based output out of metadata entirely

Instead of a metadata key, use a special syntax line in the recipe body (e.g., `= yield: 500%ml`). This was rejected because the information is declarative metadata about the recipe, not a step instruction. YAML front matter is the right place for it.

### Keep `yield` as a synonym for servings

This preserves backward compatibility for recipes using `yield: 4` to mean "serves 4." It was rejected because it leaves the core ambiguity unresolved: a parser cannot tell whether `yield` refers to people or to measurable output without inspecting the value's shape, and authors cannot express both concepts on the same recipe without one of the keys taking on a new name. The migration cost is small — `yield: 4` becomes `servings: 4` — and implementations can warn during a transition period.

## Acknowledgments

Thanks to the Cooklang community for feedback on unit-based scaling, and to @tmlmt for pointing out that `yield` already has a precise English meaning that resolves the ambiguity without adding a new key.
