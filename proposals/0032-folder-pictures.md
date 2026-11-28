# Folder pictures

* Proposal: [0032-folder-pictures](0032-folder-pictures.md)
* Authors: [Alexey Dubovskoy](https://github.com/dubadub)
* Review Manager: TBD
* Status: **Accepted, in implementation**

## Introduction

This proposal lets a folder of recipes have a picture of its own: a file named `folder` with an image extension, inside that folder. Applications that show folders as cards, rows or tiles can show it there instead of a generic folder icon.

```sh
Desserts/
  folder.jpg      ← the picture of the "Desserts" folder
  Crêpes.cook
  Crêpes.jpg      ← the picture of the "Crêpes" recipe, as today
```

Requested in [cooklang/cookcli#846](https://github.com/cooklang/cookcli/issues/846).

## Motivation

**Folders all look the same.** A recipe has a picture by naming convention (`Crêpes.jpg` next to `Crêpes.cook`). A folder has nothing. A collection organised by course (Desserts, Mains, Drinks) opens on a row of identical folder icons, and nothing in the collection can change that.

**The name has to be shared.** Several applications read the same folders: the command-line tool and its web server, mobile apps, editors, static sites built from a collection. If one application picks a name on its own, the others either copy it later without a written rule or pick a different one, and the picture works in one place only.

**The file is already handled, just not shown.** With CookCLI 0.41.1, a `folder.jpg` in a sub-folder is served by `cook server` and copied by `cook build`, and is shown nowhere. The one place it already has a meaning is next to a recipe called `folder.cook`, where it is that recipe's title picture.

**The name is already in use for this.** `folder.jpg` is the folder picture in common media tools: Windows Media Player writes `Folder.jpg` into music folders and Explorer shows it as the folder's thumbnail; Kodi reads `folder`, `cover` or `thumb` as a folder image; Plex and Jellyfin read `folder.jpg` as artwork. People who keep files in folders may already have one.

## Proposed solution

Add a paragraph to "Adding Pictures" in `conventions.md`:

> A folder can have a picture of its own: a file named `folder` with one of the picture extensions, inside that folder. Applications that show a folder as a card, row or tile may show this picture.

Nothing else changes: no grammar, no metadata.

## Detailed design

1. **Name.** The file name is `folder` plus a picture extension. The name and the extension are matched in any ASCII case, so `Folder.jpg` and `folder.JPG` count.

2. **Extensions and order.** The same extensions as recipe pictures, in this order of preference: `jpg`, `jpeg`, `png`, `webp`, `heic`. When a folder has several, the first in that order is the picture. An application that cannot draw a format (a web page and `heic`) skips it and takes the next one.

   This proposal also widens the extension list in "Adding Pictures" from `.png` and `.jpg` to these five, which is what applications already read for recipe pictures.

3. **What it is for.** An application that shows a folder as a card, row or tile may show the picture there. The folder's name, its sort order and its contents do not change. With no folder picture, the application shows what it shows today; this proposal does not ask for a fallback such as borrowing a recipe's picture.

4. **A recipe called `folder`.** `folder.jpg` next to `folder.cook` stays that recipe's title picture, as today, and is also the folder's picture. One file, two uses. `folder.1.jpg` is a step picture of `folder.cook` and never a folder picture.

5. **The collection's top folder.** The same rule applies there: a `folder.jpg` at the top of a collection is the collection's picture, for applications that show a collection as a whole (a shared cookbook, the header of a site built from it). No application has to use it.

6. **Hidden files.** A file whose name starts with `.` is not a folder picture (`.folder.jpg`), in the same way such files are never recipe pictures. A hidden attribute set by the operating system (Windows Media Player sets one on `Folder.jpg`) does not matter: the file is still read.

7. **Name only.** There is no metadata key for a folder picture. A folder has no file of its own to hold metadata, and adding one is a larger change (folder titles, descriptions and sort order would want it too).

8. **Copying collections.** Tools that copy a collection (sync, static site builds, export, import of an archive) keep `folder.*` like any other picture. Tools that report pictures not matching any recipe do not report it.

### Examples

```sh
Mains/                 no picture: a folder icon, as today
  Curry.cook

Drinks/
  Folder.JPG           the picture (case does not matter)
  Lemonade.cook

Bakes/
  folder.png
  folder.heic          a web page skips heic and shows folder.png;
                       an app that draws heic still prefers folder.png (png comes first)

Odd/
  folder.cook
  folder.jpg           the title picture of "folder" and the picture of "Odd"
```

## Effect on applications which use Cooklang

Nothing breaks when one application shows folder pictures and another does not. Each can adopt it on its own schedule.

### CookCLI (terminal and web-server)

`cook server` shows the picture on folder cards in the recipe list, in the same banner as a recipe card, and keeps the folder icon when there is none. The file is already served, including to read-only guests, and `cook build` already copies it, so no serving change is needed. A pull request implementing this exists ([cooklang/cookcli#847](https://github.com/cooklang/cookcli/pull/847)); under this proposal it would also match the name in any case.

The terminal commands do not change.

### Mobile applications

The recipe list shows the picture on a folder's row or tile, cropped like a recipe's picture. Choosing or uploading a folder picture from the app is not part of this proposal and can follow later.

## Alternatives considered

- **`cover.jpg`.** As common as `folder.jpg` in media tools, and reads better for a cookbook. Not taken because `folder.jpg` is what the existing request and implementation use, and a recipe called "Cover" is more likely than one called "Folder". Accepting both would mean two names with an order between them for every application to learn.
- **A picture named after the folder** (`Desserts/Desserts.jpg`, or `Desserts.jpg` next to the folder). Avoids a fixed English word, but collides with a recipe of the same name, which is common (`Pancakes/Pancakes.cook`), and the second form breaks when a folder is renamed and its picture is not.
- **A hidden file** (`.folder.jpg`). Stays out of file listings, but hidden files are easily lost when a collection is copied, synced or zipped, and nobody would find the convention by looking.
- **A metadata key** in a folder settings file. There is no such file; see rule 7.
- **The first recipe's picture as an automatic folder picture.** Needs no new file, but the choice of "first" is arbitrary and changes as recipes are added. Left to applications as an optional fallback.

The cost of `folder`: it is an English word in a format used in many languages. It is a file name, not text shown to anyone.

## Acknowledgments

Thanks to [SebastienSyd](https://github.com/SebastienSyd) for the request, the screenshots and the implementation in CookCLI.
