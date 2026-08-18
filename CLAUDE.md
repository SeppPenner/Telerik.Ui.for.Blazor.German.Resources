# Project rules for Claude

## What this is

Telerik.Ui.for.Blazor.German.Resources is a **data repository**, not a software project. It holds
the German translation of the localization messages of
[Telerik UI for Blazor](https://www.telerik.com/blazor-ui), one folder per Telerik version, from
`2.5.1` up to `5.1.1`. There is no solution, no `.csproj`, no source code, no test project and no
CI configuration in the repository. Nothing here compiles, `dotnet build` has nothing to build.

Consumers copy the `TelerikMessages.resx` of the version they use into their own Blazor project and
wire it up as described in
[Telerik's localization docs](https://docs.telerik.com/blazor-ui/globalization/localization).

Layout:

- **53 version folders**, named after the Telerik release they belong to (`2.5.1`, `2.6.0`, ...,
  `5.1.1`). Each contains exactly two files:
  - `TelerikMessages.resx`: the actual payload, a standard .NET resx with the German values.
  - `Strings.txt`: the plain list of the resource keys of that same resx, one per line, in the same
    order. It exists so that two versions can be diffed with a text tool without wading through XML.
- **52 `Changes_<from>_to_<to>.md`** files at the repository root, one per version step. Each has
  the heading `# Changes from version <from> to <to>`, then a `## Removed items` and a
  `## New items` section, each listing `  * <key>` entries or `  * None`.
- `README.md`: the only user documentation. Badges, a usage pointer, and a link list of every
  `Changes_*.md` file, newest on top, everything below the newest one wrapped in a
  `<details><summary>Older versions</summary>` block.
- `Changelog.md`, `License.txt` (MIT), `.gitattributes` (the stock Visual Studio template, every
  rule below `* text=auto` commented out) and `.gitignore` (a Visual Studio ignore list that matches
  nothing in this repository).

## Adding a new Telerik version

There is no build and no test suite, so "verifying a change" means checking the data itself. The
steps for a new upstream version `X.Y.Z`:

1. Fetch the German resx of that version. The upstream source is
   [telerik/blazor-ui-messages](https://github.com/telerik/blazor-ui-messages), whose `de-DE` file
   is what the folders here are based on.
2. Create the folder `X.Y.Z` and put `TelerikMessages.resx` in it.
3. Generate `Strings.txt` from that resx. It is the list of the `<data name="...">` attributes in
   file order, nothing else:

   ```powershell
   $names = ([xml](Get-Content 'X.Y.Z/TelerikMessages.resx' -Raw)).root.data.name |
       Where-Object { $_ -notin 'Bitmap1', 'Color1', 'Icon1', 'Name1' }
   Set-Content 'X.Y.Z/Strings.txt' -Value ($names -join "`r`n") -Encoding utf8NoBOM -NoNewline
   ```

   The four excluded names come from the example XML inside the resx header comment, they are not
   real entries. See the quirks below. The `-join` is what produces the CRLF line endings, and
   `-NoNewline` is what keeps the last line without one. Piping the names into `Set-Content`
   directly gets both wrong. Verified against `5.1.1`, the snippet reproduces its `Strings.txt`
   byte for byte.
4. Write `Changes_<previous>_to_X.Y.Z.md`. The content is the diff of the two `Strings.txt` files,
   nothing invented:

   ```powershell
   $a = Get-Content '<previous>/Strings.txt'
   $b = Get-Content 'X.Y.Z/Strings.txt'
   'New items:';     Compare-Object $a $b | Where-Object SideIndicator -eq '=>' | ForEach-Object InputObject
   'Removed items:'; Compare-Object $a $b | Where-Object SideIndicator -eq '<=' | ForEach-Object InputObject
   ```

   An empty section is written as `  * None`, never left out.
5. Add the link block for the new file at the top of the `Changes from ...` list in `README.md` and
   move the previously newest block into the `<details>` section.
6. Add the changelog entry, commit, tag, push. See "Releasing".

Checks that are worth running after any change to the data:

| Check | How |
| --- | --- |
| `Strings.txt` matches its resx | compare against the `<data name>` list, order included |
| The `Changes_*.md` matches the two `Strings.txt` | `Compare-Object` as above |
| Line endings and BOM survived the edit | CRLF everywhere, BOM only in the resx |
| README links resolve | every `master/Changes_*.md` link points at a file that exists |

All four checks were run over the whole repository on 2026-08-18: the 53 `Strings.txt` files match
their resx, and all 52 change lists match the data and carry the right heading. Three change lists
and one file name had to be corrected for that, see the commits of that day. A fresh finding in one
of these checks is therefore a new error, not old baggage.

## File conventions

- **`TelerikMessages.resx`**: UTF-8 **with** BOM, CRLF, two space indentation, standard resx schema
  header. German values with real umlauts (`Löschen`, `Ein Jahr wählen`, `Zur nächsten Periode`).
- **`Strings.txt`**: UTF-8 **without** BOM, CRLF, and **no trailing newline** on the last line. Keys
  are sorted case insensitively, which is why `Aggregate_Average` comes before
  `AIPrompt_OutputView_ButtonText`. That is the order the resx uses, and `Strings.txt` copies it.
- **`Changes_*.md`**: UTF-8 without BOM, CRLF, no trailing newline. Two spaces before the `*` of a
  list item.
- **Markdown at the root** (`README.md`, `Changelog.md`) is English.
- **No em dashes or en dashes** (`—`, `–`) anywhere.

## Known quirks

Do not silently "clean up" these, they are existing behaviour:

- **Tag names collide with folder names.** Every tag from `2.15.0` on is also a directory, so
  `git log 5.1.1` fails with `ambiguous argument`. Use `git log refs/tags/5.1.1` or
  `git log 5.1.1 --`.
- **Tags do not cover the early versions.** The first four tags are `1.0.2` to `1.0.5`, from the
  time when the repository had its own version number. From `2.15.0` on, the tag equals the Telerik
  version. The folders `2.5.1` through `2.14.1` therefore have no tag of their own.
- **Two tags are two part.** `2.20` and `2.21` instead of `2.20.0` and `2.21.0`, while the matching
  folders and `Changes_*.md` files use the three part form. Do not rename them, the tags are
  published.
- **All tags are lightweight.** `git cat-file -t <tag>` returns `commit`. Create new ones the same
  way, plain `git tag <version>`, no `-a`, no `v` prefix.
- **Four phantom keys in every resx.** `Bitmap1`, `Color1`, `Icon1` and `Name1` appear as
  `<data name="...">` in the schema documentation comment at the top of every resx file. A plain
  grep or an XML query over `<data>` picks them up. They are not resource entries and they are not
  in `Strings.txt`.
- **The changelog does not cover the oldest folders.** `2.5.1`, `2.6.0`, `2.6.1`, `2.7.0`, `2.7.1`,
  `2.10.0` and `2.11.0` have no line of their own, they fall under the `1.0.0.0 (2020-01-17)`
  release entry from the time the repository had its own version number. `2.8.0` and `2.9.0` share
  one line, everything from `2.12.0` on has its own. Do not invent lines for the early ones. The
  format is
  `* **Version 4.1.0.0 (2023-03-23)**: Added version 4.1.0.`, four part version in the text, no
  space before the colon.
- **AppVeyor badge without CI in the repository.** `README.md` links an AppVeyor build that is
  configured outside of this repository. There is no `.github` folder and no pipeline file here.
- **`.gitignore` fits a C# project, not this one.** `bin`, `obj/`, `packages/`, `*.pdb` and so on,
  none of which occurs here. It is inherited from the author's project template, harmless.
- **The repository is no longer maintained.** The newest folder is `5.1.1` from March 2024, and no
  further version is added, because the licensed Telerik UI for Blazor packages the resource files
  come from are not available to the author anymore. The `README.md` says so at the top. Treat
  "Adding a new Telerik version" above as documentation of how it used to work, not as a task that
  is waiting to be done.

## Releasing

1. Make the change, usually a new version folder plus its `Changes_*.md` and the `README.md` link.
2. Add an entry at the top of the list in `Changelog.md` in the existing format:
   `* **Version 5.2.0.0 (2026-08-18)**: Added version 5.2.0.`
3. Commit. The existing commit messages for this case read `Added version 5.1.1.`
4. Tag the commit with the plain Telerik version, no `v` prefix, lightweight (`git tag 5.2.0`).
5. Push the commit and the tag.

There is no package, no installer and no build output, so the release ends with the push.

## Git

- **Never amend a commit.** No `git commit --amend`, not for a typo in the message, not to add a
  forgotten file, not even when the commit is still local. Write a follow-up commit instead. The
  released versions are tags on exact commits, an amended commit leaves its tag pointing at a commit
  that is no longer in the branch.
- The tag and folder name collision above applies to every git command that takes a revision.

## Writing style

- Commit messages are written **in English only**: short, precise subject line, explanatory body
  when needed.
- Documentation in the repository is English.
- **No em dashes or en dashes** (`—`, `–`), neither in prose, commit messages nor documentation. Use
  a regular hyphen, comma, colon, parentheses or a separate sentence.
- German texts (chat replies, the resource values themselves) always use real umlauts and ß, never
  ASCII transliterations such as `ae`, `oe`, `ue` or `ss`. Resource keys stay unchanged.
