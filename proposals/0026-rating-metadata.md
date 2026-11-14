# `rating` metadata

* Proposal: [0026-rating-metadata](0026-rating-metadata.md)
* Authors: [Alexey Dubovskoy](https://github.com/dubadub)
* Review Manager: TBD
* Status: **Awaiting review**

## Introduction

This proposal adds one key to the canonical metadata: `rating`, the recipe keeper's own rating of the recipe, a number from 1 to 5. With an agreed key and scale, applications can show it as stars, and the rating stays with the recipe when the file moves from one application to another.

```cooklang
---
title: Pumpkin cheesecake
rating: 4
---
```

## Motivation

**There is no agreed key.** The canonical metadata table has `difficulty`, `cuisine`, `diet` and `tags`, and nothing for a rating. People who rate their recipes pick their own key and shape. Public `.cook` files on GitHub show `stars: 2`, `rating: 4.5` and an empty `rating:` field, and a community member recently asked whether `rating: 4/5` could be shown as "★★★★☆".

**Applications cannot do anything with it.** A key an application does not know is at best printed as text. Nothing can draw stars, sort a collection by them, or keep the rating when a recipe is opened in another application.

**The same value is treated differently depending on how it is typed.** With CookCLI 0.38.1:

| Front matter | Parsed as | `cook recipe` | `cook server` recipe page |
| --- | --- | --- | --- |
| `rating: 4` | number | prints `rating: 4` | nothing shown |
| `rating: 4.5` | number | prints `rating: 4.5` | nothing shown |
| `rating: 4/5` | text | prints `rating: 4/5` | a plain `rating: 4/5` label |
| `rating: "★★★★☆"` | text | prints it | a plain label |

## Proposed solution

Add one row to the canonical metadata table in `conventions.md`:

| Key | Purpose | Example value |
| --- | --- | --- |
| `rating` | The recipe keeper's own rating of the recipe, a number from 1 to 5. Higher is better. | `4`, `4.5` |

An application that knows the key shows `rating: 4` as "★★★★☆".

## Detailed design

1. **Whose rating.** It is the rating given by whoever keeps the file. It is not the average from the website the recipe came from, and not a count of votes.

2. **Scale.** Fixed: 1 to 5, higher is better. A fixed scale lets two applications show the same file the same way without either of them converting.

3. **Value.** A number. Decimals are allowed (`4.5`, `3.7`). An application that draws stars rounds to the nearest half.

4. **Not rated.** A missing key, an empty value and `0` all mean the recipe is not rated. None of them is an error. `0` is included because an application that clears a rating may write it.

5. **`N/5`.** Applications should also read the text `N/5` as the number N: `rating: 4/5` is 4, `rating: 4.5/5` is 4.5. People write it this way and it reads well as plain text. The number is the form applications write.

6. **Anything else is not a rating.** A number outside 1 to 5 (other than `0`), another denominator (`8/10`) and any other text (`"★★★★☆"`, `great`) are not a rating. An application shows the raw value as text, as it does for an unknown key today, and a linter should report it. An application must not rescale: `8/10` is not shown as four stars.

7. **One key.** There are no aliases. `stars`, `score` and `rate` are ordinary custom keys.

8. **Display.** Filled and empty stars out of five: 4 is "★★★★☆". A half is drawn as a half star where the application draws its own icons, and written "★★★★½" in plain text. Assistive technology gets the words, for example "4 out of 5".

9. **Shared and published recipes.** The rating is in the file, so it travels with the file: through sync, a shared link, a public repository. An application that publishes recipes may show it as the author's own rating. It must not present it as an average of several people's ratings, and must not put it in structured data as an aggregate rating.

10. **Importers.** A tool that converts a web page or a photo into a `.cook` file does not write `rating`. The rating on the source page is other people's, and the keeper has not cooked the recipe yet.

No grammar changes. The value is ordinary YAML front matter.

### Canonical test

One test for the metadata result, to be added with the conventions change:

```yaml
testMetadataRating:
  source: |
    ---
    rating: 4.5
    ---
  result:
    steps: []
    metadata:
      "rating": 4.5
```

## Effect on applications which use Cooklang

Parsers that expose typed canonical keys add `rating` and return a number from 1 to 5 or nothing. Parsers that only pass metadata through need no change.

One thing to watch in applications that list unknown keys generically: once the parser treats `rating` as canonical, it is no longer among the unknown keys. An application has to start showing the rating itself in the same release, including the raw text for a value that is not a rating, or the key disappears from the page.

### CookCLI (terminal and web-server)

`cook recipe` prints the rating as stars next to the other metadata. The web server's recipe page shows a star label next to `difficulty`, `diet` and the rest. A value that is not a rating is shown as plain text.

No new sub-command. Sorting or filtering the recipe list by rating is useful and can follow; it is not part of this proposal.

### Mobile applications

The recipe screen shows the stars with the other metadata. Setting a rating by tapping the stars, which writes the number to the file, and sorting the recipe list by rating can follow; they are not required by this proposal.

## Alternatives considered

**A free scale** (`8/10`, or nested `value` and `best` keys). It lets each author keep their habit, but every application then has to convert between scales, and a 7 means little without its upper bound. Five-point ratings are what recipe applications and websites commonly show.

**Whole numbers only.** Simpler, but half stars are common, and a file with `4.5` already exists in public repositories.

**Only the `N/5` text form.** It reads well, but it is text in YAML, and `4/5` is a fraction (0.8) everywhere else in Cooklang. Reading it as an alternative spelling of the number costs little; making it the only form would make every writer produce text where a number does.

**Aliases such as `stars`.** Each alias is one more key every application has to know, for a field with almost no existing files to migrate.

**Leave it to `tags`** (`tags: [5-stars]`). It works today, but no application can tell that tag from any other, so there are no stars and no ordering.

**A rating per cook or per date.** That is a cooking log, a different and larger feature. A single current rating is what most people keep.

## Acknowledgments

Thanks to the community member who asked whether CookCLI could show `rating: 4/5` as stars and offered to implement it.
