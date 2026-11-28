# `rating` metadata

* Proposal: [0026-rating-metadata](0026-rating-metadata.md)
* Authors: [Alexey Dubovskoy](https://github.com/dubadub)
* Review Manager: TBD
* Status: **Awaiting review**

## Introduction

This proposal adds one key to the canonical metadata: `rating`, the recipe keeper's own rating of the recipe. It is a number out of 5, or out of another scale when the author writes one: `rating: 4` is 4 out of 5, `rating: 3/10` is 3 out of 10. With an agreed key and form, every application can read the rating and show it in its own way, and the rating stays with the recipe when the file moves from one application to another.

```cooklang
---
title: Pumpkin cheesecake
rating: 4
---
```

## Motivation

**There is no agreed key.** The canonical metadata table has `difficulty`, `cuisine`, `diet` and `tags`, and nothing for a rating. People who rate their recipes pick their own key and shape. Public `.cook` files on GitHub show `stars: 2`, `rating: 4.5` and an empty `rating:` field, and a community member recently asked whether `rating: 4/5` could be shown as "★★★★☆".

**Applications cannot do anything with it.** A key an application does not know is at best printed as text. Nothing can show it as a rating, sort a collection by it, or keep it when a recipe is opened in another application.

**People rate on different scales.** Five stars is the most common, but some people score out of 10 and some keep only a few levels. A key that accepts one scale makes everyone else convert their ratings before the key is of any use to them.

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
| `rating` | The recipe keeper's own rating of the recipe: a number out of 5, or `N/M` for N out of M. Higher is better. | `4`, `4.5`, `8/10` |

An application that knows the key shows `rating: 4` as 4 out of 5, in whatever form suits it: "★★★★☆", "⭐ 4/5" or a filled bar.

## Detailed design

1. **Whose rating.** It is the rating given by whoever keeps the file. It is not the average from the website the recipe came from, and not a count of votes.

2. **Form.** The value is `N` or `N/M`.
   - `N` is N out of 5. The default scale is 5.
   - `N/M` is N out of M. M is a whole number, 1 or more. `4/5` is the same rating as `4`.

   N can have decimals (`4.5`, `7.5/10`) and runs from 0 to M, both included. Higher is better. YAML reads `4` as a number and `4/5` as text; both are accepted.

3. **Zero is a rating.** `0` is the lowest rating, not the absence of one: `0` is 0 out of 5 and `0/10` is 0 out of 10.

4. **Not rated.** A missing key and an empty or `null` value mean the recipe is not rated. So does any value that does not have the form above: a number above its scale (`7`, `7/5`), a negative number, a scale that is not a whole number of 1 or more (`3/0`, `3/4.5`), and any other text (`"★★★★☆"`, `great`). None of these is an error in the recipe. An application shows no rating for them. It may still show the raw value as plain text, as it does for a custom key, and a linter should report it, because the author most likely meant to write a rating.

5. **The scale is never guessed.** A bare number is always out of 5. `rating: 8` is not read as 8 out of 10: it is above the default scale, so the recipe is not rated. A rating on another scale writes its scale.

6. **Not a fraction.** Elsewhere in Cooklang `4/5` is the fraction 0.8. In `rating`, `N/M` gives a score and its scale: `3/10` is 3 out of 10, not the number 0.3.

7. **Display.** How a rating is shown is up to the application: icons, a number with its scale ("8/10", "⭐ 4.5/5"), a bar, or anything else. Whatever the form, the scale has to be visible, because a 7 means little without it. An application that always draws the same number of icons may convert for display, showing `8/10` as four of five stars, but must not write the converted value back. Assistive technology gets the words, for example "4 out of 5".

8. **Comparing.** Ratings on different scales compare by N divided by M, so `4`, `4/5` and `8/10` are equal. An application that sorts or filters by rating compares them this way.

9. **Writing.** An application that changes a rating keeps the scale already in the file: changing `8/10` writes another value out of 10. For a recipe with no rating, it writes a bare number on the default scale, or `N/M` on the scale it offers. Clearing a rating removes the key or leaves the value empty.

10. **One key.** There are no aliases. `stars`, `score` and `rate` are ordinary custom keys.

11. **Shared and published recipes.** The rating is in the file, so it travels with the file: through sync, a shared link, a public repository. An application that publishes recipes may show it as the author's own rating. It must not present it as an average of several people's ratings, and must not put it in structured data as an aggregate rating.

12. **Importers.** A tool that converts a web page or a photo into a `.cook` file does not write `rating`. The rating on the source page is other people's, and the keeper has not cooked the recipe yet.

No grammar changes. The value is ordinary YAML front matter.

### Canonical tests

Two tests for the metadata result, to be added with the conventions change. As for every other key, the parser passes the value through as written; reading it as a rating is up to the application.

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

testMetadataRatingWithScale:
  source: |
    ---
    rating: 3/10
    ---
  result:
    steps: []
    metadata:
      "rating": "3/10"
```

## Effect on applications which use Cooklang

Parsers that expose typed canonical keys add `rating` and return the value and its scale (5 when none is written), or nothing when the recipe is not rated. Parsers that only pass metadata through need no change.

One thing to watch in applications that list unknown keys generically: once the parser treats `rating` as canonical, it is no longer among the unknown keys. An application has to start showing the rating itself in the same release, including the raw text for a value that is not a rating, or the key disappears from the page.

### CookCLI (terminal and web-server)

`cook recipe` prints the rating with its scale next to the other metadata, for example `rating: 4/5` or `rating: 8/10`. The web server's recipe page shows a rating label next to `difficulty`, `diet` and the rest, for example "⭐ 4.5/5". A value that is not a rating is shown as plain text.

No new sub-command. Sorting or filtering the recipe list by rating is useful and can follow; it is not part of this proposal.

### Mobile applications

The recipe screen shows the rating with the other metadata, in the application's own style. Setting a rating from the application, which writes it to the file on the file's scale, and sorting the recipe list by rating can follow; they are not required by this proposal.

## Alternatives considered

**A fixed 1 to 5 scale.** This was the first draft of the proposal. Two applications show the same file the same way without converting, but anyone who rates out of 10 or out of 3 has to convert every rating first, and the key is tied to five-star display. With the scale written in the value, a five-point rating stays as short as before (`4`), and another scale costs a few more characters (`8/10`).

**Guessing the scale from the value,** for example reading a bare number above 5 as out of 10. It saves typing `/10`, but `rating: 5` is then 5 out of 5 or 5 out of 10, and two applications can guess differently. A bare number is always out of 5.

**Capping the scale,** for example at 10. It would rule out scores out of 100, which some people keep, and buys nothing: dividing N by M compares any two scales.

**Nested keys** (`rating: {value: 8, best: 10}`). The same information as `8/10`, but longer to type and harder to read as plain text.

**Whole numbers only.** Simpler, but half stars and scores such as 7.5 out of 10 are common, and a file with `4.5` already exists in public repositories.

**Zero means not rated.** The first draft did this, so that an application clearing a rating could write `0`. But 0 is a real score on many scales, and clearing a rating can remove the key instead.

**Aliases such as `stars`.** Each alias is one more key every application has to know, for a field with almost no existing files to migrate.

**Leave it to `tags`** (`tags: [5-stars]`). It works today, but no application can tell that tag from any other, so there is no rating to show and no ordering.

**A rating per cook or per date.** That is a cooking log, a different and larger feature. A single current rating is what most people keep.

## Acknowledgments

Thanks to the community member who asked whether CookCLI could show `rating: 4/5` as stars and offered to implement it, and to the reviewer on the pull request who pointed out that a fixed scale is limiting.
