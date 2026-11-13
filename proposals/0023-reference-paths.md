# Reference path resolution

* Proposal: [0023-reference-paths](0023-reference-paths.md)
* Authors: [Alexey Dubovskoy](https://github.com/dubadub)
* Review Manager: TBD
* Status: **Awaiting review**
* Previous Proposal: [0015-other-recipes](0015-other-recipes.md)

## Introduction

A recipe reference is a path, and the specification does not say clearly where that path starts. This proposal defines it with three forms: `/path` is read from the root of the recipe collection, `./path` from the folder of the file that writes the reference, and `../path` from the parent of that folder. The same rule applies wherever a path to a recipe is written: in recipes, in `.menu` files and in `.shopping-list` files. References written under today's rule keep working during a transition period.

Discussion thread: [References to other recipes are interpreted as relative to pwd](https://github.com/cooklang/spec/discussions/139)

## Motivation

[Proposal 0015](0015-other-recipes.md) introduced references and showed both `@./Pesto Sauce{}` and `@../sauces/Bechamel{}` in its examples, but never said what the path is relative to. The specification later settled on one sentence: "The path is relative to the root of the recipes directory". That leaves three problems.

**`./` does not mean what it means everywhere else.** In a shell, in Markdown, in HTML and in most programming languages `./` is "next to this file". In Cooklang it is "the top of the collection". A recipe in `Breakfast/` that uses the salsa sitting beside it has to write `@./Breakfast/Salsa{}`.

**`../` is undefined, and implementations disagree.** Read from the root, `../x` always points outside the collection, so it cannot mean anything. Today:

| Implementation | `./path` | `../path` |
|---|---|---|
| Specification | from the root | not mentioned |
| cooklang-rs parser | marked as a reference, not resolved | marked as a reference, not resolved |
| [cooklang-find](https://github.com/cooklang/cooklang-find) (mobile apps, desktop editor) | from the root | resolves to nothing |
| [CookCLI](https://github.com/cooklang/cookcli) since 0.37.0 | from the root | from the folder of the writing file |

So a recipe that uses `../` builds a correct shopping list in CookCLI and shows a broken reference in the mobile apps.

**A folder of recipes cannot be moved or shared as a unit.** Every reference spells the whole path from the root, so a folder with a dish and its three components stops working when it is renamed, moved one level down, or copied into someone else's collection.

There is also no way to say "from the root" on purpose, as distinct from "from here". An author with a `Shared/` folder of stocks and sauces wants the first; an author with a self-contained folder wants the second. Today they share one spelling.

## Proposed solution

A recipe reference starts in one of three ways, and the start says where the path begins:

| Written | Read from |
|---|---|
| `/path` | the root of the recipe collection |
| `./path` | the folder of the file that writes the reference |
| `../path` | the parent of that folder |

In `Breakfast/Burrito.cook`:

```cooklang
Warm @/Shared/Red Beans{200%g} and spoon over the tortilla.
Top with @./Salsa{2%tbsp} and @../Sauces/Crema{1%tbsp}.
```

- `@/Shared/Red Beans{}` is `Shared/Red Beans.cook`, wherever the recipe that writes it lives.
- `@./Salsa{}` is `Breakfast/Salsa.cook`, the file beside the burrito.
- `@../Sauces/Crema{}` is `Sauces/Crema.cook`, one folder up and into `Sauces/`.

`@/…{}` is new syntax. `@./…{}` and `@../…{}` exist; this proposal changes what the first means for files that are not at the root and defines the second.

## Detailed design

### Grammar

An ingredient whose name starts with `/`, `./` or `../` and does not end with `/` is a recipe reference. The first form is new: today `@/Shared/Red Beans{}` is an ordinary ingredient named `/Shared/Red Beans`.

Everything else about references is unchanged: the quantity and the [scaling rules](../conventions.md#scaling-referenced-recipes), the `?` modifier from [proposal 0018](0018-optional-components.md) (`@?/Sauces/Chimichurri{}`), the `.cook` extension being left out, and the name shown to the reader being the last segment of the path. A bare name such as `@Pesto{}` stays an ingredient.

Parsers that accept `.\` and `..\` keep reading `\` as `/` in those two forms. A leading `\` does not start a reference.

### Resolution

Let *root* be the folder that holds the recipe collection and *folder* the folder of the file that writes the reference.

1. Take the starting point from the form: *root* for `/path`, *folder* for `./path` and `../path`.
2. Split the path on `/`. Drop empty segments and `.` segments. A `..` segment steps to the parent; it may appear anywhere in the path and may repeat (`../../Basics/Stock`).
3. If a `..` would step above *root*, the reference resolves to nothing. An application never opens a file outside the collection because a recipe pointed at it.
4. Look the result up as today: a name ending in `.cook` or `.menu` is looked up as written; any other name is tried as `<name>.cook` and then `<name>.menu`.

**The base is the file being read.** When recipe A references recipe B and B references C, the path to C is read from B's folder, not from A's.

**The root** is the folder the application was opened on: the recipes folder in a mobile app, the workspace folder in an editor, the base path (or the current directory) in a command-line tool. A leading `/` never means the root of the file system.

**A reference that does not resolve is marked, not fatal.** Applications show it as broken and carry on with the rest of the recipe; validation tools report it.

### Menus

A `.menu` file is a Cooklang file, so the rule applies as written, with the menu's own folder as *folder*. `conventions.md` currently says "Recipe paths are relative to the root recipe directory"; that sentence is replaced by a pointer to this rule.

### Shopping lists

In a [`.shopping-list`](0016-shopping-list-format.md) file a recipe line may start with `/`, `./` or `../`. Every recipe line, at any nesting level, is read from the folder that holds the list.

This changes one sentence of `conventions.md`, added with [proposal 0018](0018-optional-components.md), which says an accepted optional recipe reference is "written with its path as in the recipe". Copied verbatim, `./Salsa` from `Breakfast/Burrito.cook` would point at a different file once it sits in a list at the root. A writer therefore **rebases** a reference when it copies it from a recipe into a list: it resolves the reference, then writes the path from the list's folder. Matching a list line back to a reference in the recipe compares resolved paths, not text.

Writers should keep writing list lines as `./path`. For a list at the root, which is where applications keep it, that is what is written today, it means the same file under the old and the new rule, and applications that predate this proposal keep reading it.

### Links in notes

[Proposal 0022](https://github.com/cooklang/spec/pull/159) resolves a link path in a note "the same way as the path of a recipe reference". If both proposals are accepted, a link takes this rule, and a leading `/` in a link is the collection root.

### Backward compatibility

Recipes at the root of a collection are not affected: for them *folder* and *root* are the same folder, so `./path` means what it always did.

The meaning changes for one group of references: `./path`, with no `..` in it, written in a file that is not at the root. Those were written to be read from the root. To keep them working, resolution gets a fallback for exactly this group:

1. Look in the folder of the writing file. If a recipe is there, that is the answer.
2. If nothing is there, look for the same path from the root. If a recipe is there, the reference resolves, and the resolver reports it as a **legacy** reference.
3. Otherwise the reference is broken.

The fallback covers recipe lines in a `.shopping-list` that is not at the root in the same way.

Applications that can show warnings report legacy references and offer to rewrite them to the root form: `@./Shared/Red Beans{}` becomes `@/Shared/Red Beans{}`. This changes one character, keeps the meaning, and the result no longer depends on where the file sits. Applications must not rewrite a user's files without being asked.

The fallback has a cost. While it exists, adding `Breakfast/Shared/Red Beans.cook` changes what an old `@./Shared/Red Beans{}` in `Breakfast/Burrito.cook` points at, from the file at the root to the new one beside it. This is why legacy references are reported instead of being resolved silently. The fallback is meant to be removed in a later revision of the specification, once the tools that rewrite legacy references have been available for a while.

**Old applications reading new files.** A collection is often read by several applications that update at different times:

| Written in a recipe | An application that predates this proposal |
|---|---|
| `@/Shared/Red Beans{}` | An ingredient named `/Shared/Red Beans`. It appears on the shopping list as an item to buy, and the referenced recipe's ingredients are not included |
| `@./Salsa{}` in `Breakfast/`, meaning the file beside it | Looked up from the root: not found, or a different recipe if the root has a `Salsa.cook` |
| `@../Sauces/Crema{}` | Resolves in CookCLI 0.37.0 and later; not found in applications built on cooklang-find |
| An old `@./Shared/Red Beans{}`, left as it is | Resolves everywhere, old and new |

For this reason applications should read the new forms for some time before any of them writes one, and a tool that rewrites legacy references should say that the result needs up-to-date applications.

### Parser output

The parser still does not resolve paths or open files. It reports each reference with its path as written, including whether it starts at the root, so that a resolver can apply the rules above. Resolution needs two inputs the parser does not have: the root and the folder of the file being read.

### Canonical cases

A collection with `Pesto.cook`, `Shared/Red Beans.cook`, `Breakfast/Burrito.cook`, `Breakfast/Salsa.cook` and `Plans/Week.menu`:

| Written in | Reference | Resolves to |
|---|---|---|
| `Breakfast/Burrito.cook` | `./Salsa` | `Breakfast/Salsa.cook` |
| `Breakfast/Burrito.cook` | `../Pesto` | `Pesto.cook` |
| `Breakfast/Burrito.cook` | `/Shared/Red Beans` | `Shared/Red Beans.cook` |
| `Breakfast/Burrito.cook` | `./../Shared//Red Beans` | `Shared/Red Beans.cook` |
| `Breakfast/Burrito.cook` | `./Shared/Red Beans` | `Shared/Red Beans.cook`, legacy |
| `Breakfast/Burrito.cook` | `../../Pesto` | nothing (above the root) |
| `Pesto.cook` | `./Breakfast/Salsa` | `Breakfast/Salsa.cook` (not legacy: both readings agree) |
| `Pesto.cook` | `../Pesto` | nothing (above the root) |
| `Plans/Week.menu` | `/Breakfast/Burrito` | `Breakfast/Burrito.cook` |
| `Plans/Week.menu` | `../Breakfast/Burrito.cook` | `Breakfast/Burrito.cook` |
| `Plans/Week.menu` | `./Breakfast/Burrito` | `Breakfast/Burrito.cook`, legacy |

With `Breakfast/Shared/Red Beans.cook` added to the collection, `./Shared/Red Beans` written in `Breakfast/Burrito.cook` resolves to `Breakfast/Shared/Red Beans.cook` and is not legacy.

These cases would be added to the test suite in a form that does not need a file system: a list of the files in the collection, the writing file, the reference, and the expected result.

## Effect on applications which use Cooklang

Applications need the folder of the file they are reading as well as the root when they resolve a reference. Most of the change is in the shared resolver; applications pass one more argument and handle the legacy flag.

Moving and renaming needs attention in every application that does it. Root paths and relative paths break in opposite cases:

| | Move the referencing file alone | Move a folder with both files in it | Move or rename the target |
|---|---|---|---|
| `/path` | still works | breaks | breaks |
| `./path`, `../path` | breaks | still works | breaks |

An application that moves or renames a file should rewrite the references to it, and the relative references inside it.

### CookCLI (terminal and web-server)

1. Resolve the three forms and the legacy fallback, in place of the current rule (from the root unless the path contains `..`).
2. `cook doctor validate` reports legacy references as warnings and broken references as it does today. With `./` read from the writing file, the case in the discussion thread (two recipes in one folder, validated from a parent directory) passes from any directory.
3. A command that rewrites legacy references to the root form.
4. The web server's rename and move rewrite relative references in the moved file as well as references to it.
5. The seed recipes that reference `./Shared/…` from a subfolder switch to `/Shared/…`.

### Mobile applications

1. Resolve the three forms and the legacy fallback when opening a referenced recipe, building a shopping list and showing a menu.
2. Show a reference that does not resolve as broken, without failing the recipe.
3. Meal plans and shopping lists written by the app use paths that resolve under this rule: `./path` from the list's folder in `.shopping-list`, and `/path` in menus once reading applications have been released.
4. No rewriting of existing recipes.

### Editors and the language server

1. Completion after `@./` and `@../` offers paths from the current file's folder; after `@/`, paths from the root.
2. A warning on legacy references, with a quick fix that rewrites to the root form.
3. Go-to-definition and preview links use the same resolver.

## Open questions

1. **The root in command-line tools.** `./` and `../` no longer depend on where a command is run, but `/` depends on the root. `cook` run inside `Breakfast/` takes `Breakfast/` as the root, so `@/Shared/Red Beans{}` is above it. The options are to leave this and say so in the error message, or to find the root by walking up to a marker, as suggested in the discussion thread.
2. **When the fallback is removed.** A date, a number of releases, or a specification version.
3. **Nested shopping list lines.** Rebased to the list's folder, as proposed, or copied as written and read from the folder of the parent line.
4. **What the rewrite produces.** The root form, as proposed, or the shortest relative path.

## Alternatives considered

1. **Keep `./` as the root and add nothing.** Simple, and stable when a single file moves. But it keeps `./` at odds with its meaning everywhere else, leaves `../` meaningless, and does not let a folder of recipes stand on its own. This is the current state and the reason for the discussion thread.
2. **Make `./` relative to the file with no fallback.** One clean rule from the first day, at the price of breaking every existing reference written in a subfolder, including recipes that ship with the tools, until the author rewrites them.
3. **Read `./` from the file first and the root second, permanently.** Nothing ever breaks, but `./` never gets a single meaning, and a new file beside a recipe can silently take over a reference that used to point at the root.
4. **Read only `../` from the file and keep `./` at the root.** This is what CookCLI does today. It makes `../` useful without changing `./`, but two files in one folder still cannot reference each other without the full path, and the rule ("relative to the file only if the path steps up") is hard to explain.
5. **Reference by name or title, not by path.** Raised in the discussion thread and considered in proposal 0015 as `@@name{}`. A move would no longer break references, but names must then be unique across the collection, a rename breaks references just as a move does now, and lookup needs an index of the whole collection. This proposal stays with 0015's choice of paths and only defines where a path starts.
6. **A marker file for the root.** Useful for open question 1, but it is a tooling convention, and the three forms do not depend on it.

## Acknowledgments

Thanks to [@sievertst](https://github.com/sievertst) for reporting the problem, for arguing the case for paths relative to the recipe, and for laying out the counterarguments about moving files that shaped the root form in this proposal.
