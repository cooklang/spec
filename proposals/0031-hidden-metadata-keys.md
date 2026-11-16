# Hidden metadata keys

* Proposal: [0031-hidden-metadata-keys](0031-hidden-metadata-keys.md)
* Authors: [Alexey Dubovskoy](https://github.com/dubadub)
* Review Manager: TBD
* Status: **Awaiting review**

## Introduction

This proposal adds one convention for metadata keys: a key whose name starts with a dot is hidden. Applications keep the key and may read it, and they do not show it to a person reading or cooking the recipe.

```cooklang
---
title: Tarte flambée
servings: 4
wine: Riesling
.imported: 2026-11-01
.myapp:
  id: 42
  source-hash: 9f2c
nutrition:
  kcal: 1443
  .database-id: 77
---
```

The recipe page shows `wine: Riesling` and the calories. It shows nothing for `.imported`, `.myapp` or `.database-id`.

## Motivation

**Every unknown key is shown.** Front matter takes any key, and applications show the keys they do not know next to the canonical ones. That is right for `wine: Riesling`. It is wrong for data a person or a tool keeps in the file for its own use: an id in another system, the date of an import, who last edited the file, a reference into a nutrition database.

**There is no way to keep a key and not show it.** Today the choices are to leave the line out of the file, or to put it in a comment, where no tool can read it as data.

**A dot does nothing today.** With CookCLI 0.38.1:

| Front matter | `cook recipe` | `cook server` recipe page |
| --- | --- | --- |
| `.nutrition-id: abc123` | prints `.nutrition-id: abc123` | a plain `.nutrition-id: abc123` label |
| `.count: 3` | prints `.count: 3` | nothing shown |
| `.myapp:` with a mapping under it | nothing printed | nothing shown |

The number and the mapping are missing from the page only because the server shows text values and nothing else. Once it learns to show numbers, lists and mappings, they appear like any other key.

The idea comes from a [CookCLI feature request](https://github.com/cooklang/cookcli/issues/459) about showing custom metadata, which asked for "a way to hide a family or a single entry from display (e.g. a `.`-prefixed key)".

## Proposed solution

Add one paragraph to the metadata section of `conventions.md`:

> A metadata key that starts with a dot (`.`) is hidden. Applications keep hidden keys and write them back unchanged, and leave them out of everything they show to a reader. The rule applies to keys inside a mapping as well, and a hidden key hides everything under it. A hidden key is still part of the file: it is not private.

## Detailed design

1. **The mark.** A key is hidden when it is a text key and its first character is `.`. A dot anywhere else in a key means nothing here: `source.url` and `time.prep` stay as they are.

2. **YAML numbers are not hidden keys.** A bare `.5` or `.inf` is a YAML number, not text. Such a key is not a hidden key, and some parsers reject it. Quote it (`".5"`) if that name is really wanted.

3. **Every depth.** The rule applies to a key at the top level and to a key inside a mapping. Hiding a key hides everything under it: `.myapp:` hides `id` and `source-hash`, which need no dots of their own.

4. **Not a canonical key.** `.servings` is a custom key named `.servings`. It does not set the servings and nothing is scaled by it. Hiding a canonical key while keeping its effect is not offered.

5. **What "shown" covers.** Hidden keys are left out of every view made for a reader: the recipe page, recipe cards and lists, cook mode, print, a shared or published page, the plain-text view in a terminal, and exports that are documents (Markdown, LaTeX, Typst, PDF) or structured data for a public page.

6. **What it does not cover.** The source text and every data output keep hidden keys: the `.cook` file, a text editor, JSON and YAML output, a parser's result, and the data a report template receives.

7. **Writing files.** An application that rewrites a recipe (formatting, saving a scaled copy, editing a field) writes hidden keys back unchanged and in the same place.

8. **Forms.** A form for editing metadata does not list hidden keys and does not remove them when it saves. The text of the file is the place to change them.

9. **Search and filters.** Hidden keys are not offered as filters and user-facing search does not match on them. An application may read its own hidden keys for its own work.

10. **Not private.** A hidden key is in the file. It is synced, shared, published and downloaded with the file. Nothing secret belongs in it.

11. **Keys written by tools.** A tool that stores its own data in a recipe should put it in one hidden mapping named after itself (`.myapp:`), so that two tools do not pick the same key.

12. **No warnings.** A hidden key is never reported as unknown or unsupported.

No grammar changes. Keys and values are ordinary YAML front matter.

### Canonical test

One test for the metadata result, to be added with the conventions change. A parser keeps hidden keys:

```yaml
testMetadataHiddenKey:
  source: |
    ---
    wine: Riesling
    .imported: 2026-11-01
    ---
  result:
    steps: []
    metadata:
      "wine": Riesling
      ".imported": 2026-11-01
```

## Effect on applications which use Cooklang

Parsers need no change to read such files: a hidden key is already kept like any other custom key. A parser library can add a helper that returns the custom keys without the hidden ones, nested ones included, so that every application applies the same rule. The existing way to get all custom keys should stay, because some callers use them as data.

**Files with hidden keys in applications that do not know the rule yet.** Nothing fails. A hidden key with a text value is shown with the dot in its name (`.imported: 2026-11-01`). What happens to numbers and mappings depends on the application: some skip them, some print them. A person can start writing hidden keys at once. A tool that writes hidden keys into other people's files should wait until the applications those people use have the rule.

### CookCLI (terminal and web-server)

The web server's recipe page and the default output of `cook recipe` leave hidden keys out. Markdown output leaves them out of its front matter. JSON, YAML and Cooklang output keep them.

No new sub-command or flag.

### Mobile applications

The recipe screen leaves hidden keys out of the list of other metadata. Nothing else changes.

## Alternatives considered

**An underscore (`_key`).** It is also free today. A dot is the more familiar mark, because files that start with a dot are hidden on Unix systems, and an underscore already stands for a space in key names in some applications.

**A list of keys to hide** (`hidden: [imported, id]` in the recipe or in a configuration file). It keeps key names clean, but the list and the keys can drift apart, a list in a configuration file does not travel with the recipe, and every application has to read the list before it can show anything.

**Comments** (`-- imported: 2026-11-01`). Hidden everywhere already, but a comment is not data: tools cannot read it reliably, and a formatter may move or drop it.

**A separate file next to the recipe.** It keeps the recipe clean, but the data is lost when the recipe is moved, shared or renamed without it.

**Top level only.** A simpler rule, but it leaves no way to hide one field of a mapping that is otherwise shown.

**A dot that hides a canonical key but keeps its effect** (`.servings: 4` still scales). Every application would have to strip the dot before every lookup, for a case nobody has asked for.

**Calling the keys "private".** The file still carries the key wherever it goes, so "hidden" says what the rule does and no more.

## Open questions

- **The terminal.** Should the default output of `cook recipe` hide the keys, as proposed, or stay a view that prints everything?
- **Markdown front matter.** Other tools read it as data, and someone may want an id carried over. Should Markdown export keep hidden keys?
- **Search.** Hidden keys are not searchable in this proposal. Searching by an id stored in a hidden key is a fair use; a match the page cannot explain is the cost.
- **The tool namespace.** Is the "one hidden mapping per tool" advice worth having in the conventions now, or only once several tools write hidden keys?

## Acknowledgments

Thanks to the author of the CookCLI feature request and its implementation for the idea of the dot prefix.
