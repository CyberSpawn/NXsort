# NXsort

NXsort is a Windows-friendly .NET 10 console tool that reads Nintendo Switch `.nsp` and `.xci` files, tells you what is really inside them, checks that they are intact, and (optionally) renames, sorts into folders, and pulls icons out of them.

It never trusts a filename. Everything it reports is read from the file's own metadata (the CNMT and the NACP), and the filename is then compared against that.

What it does:

- **Identifies** each file: base game, update, DLC, or a bundle of several.
- **Verifies** every content NCA by hashing it (SHA-256) and comparing against the hash recorded in the CNMT.
- **Checks filenames** against the convention `Name [TITLEID][vREV]` and warns when the title ID, version or DLC count in the name is wrong.
- **Renames** files (and, optionally, their folders) to match, with a prompt for every change.
- **Sorts** loose files into one folder per game, with updates and DLC grouped with their base game.
- **Extracts** one icon per game.
- **Remembers** what it learned in a manifest, so unchanged files aren't re-read or re-hashed next time.

---

## Contents

1. [Requirements and setup](#requirements-and-setup)
2. [Quick start](#quick-start)
3. [Command-line options](#command-line-options)
4. [How a run works](#how-a-run-works)
5. [How NXsort identifies a file](#how-nxsort-identifies-a-file)
6. [The naming convention](#the-naming-convention)
7. [Renaming rules in depth](#renaming-rules-in-depth)
8. [Standalone DLC naming](#standalone-dlc-naming)
9. [Folder renaming](#folder-renaming---rename---recursive)
10. [Folder sorting](#folder-sorting---folderize)
11. [Icons](#icons---icon)
12. [Verification](#verification)
13. [The manifest (scan cache)](#the-manifest-scan-cache)
14. [The log](#the-log)
15. [Files NXsort reads and writes](#files-nxsort-reads-and-writes)
16. [Exit codes](#exit-codes)
17. [Building](#building)
18. [Limitations](#limitations)

---

## Requirements and setup

- .NET 10 SDK (to build) or runtime (to run).
- A `prod.keys` file from your own console. NXsort needs it to decrypt the metadata and control NCAs.
- [LibHac](https://github.com/Thealexbarney/LibHac) **0.15.0**, pinned on purpose (see [Building](#building)).

Put these next to `NXsort.exe`:

| File / folder | Required | Purpose |
|---|---|---|
| `prod.keys` | Yes | Decryption keys (or point to one with `--keys`) |
| `roms/` | Default scan folder | Where your `.nsp`/`.xci` files live (or use `--roms`) |
| `custom.json` | No | Your own name overrides |
| `titles.json` | No | A title database, used only to name standalone DLC |

`custom.json`, `titles.json`, `nxsort.log`, `nxsort.manifest.json`, `prod.keys` and the `icons` folder always live next to `NXsort.exe`, even when you scan a different folder.

## Quick start

```text
NXsort                                  scan ./roms, verify everything, change nothing
NXsort --rename                         ...and offer to fix filenames
NXsort --rename --recursive             ...also look in subfolders, and check folder names
NXsort --folderize                      ...and offer to sort loose files into per-game folders
NXsort --rename --folderize --icon      the full treatment
NXsort D:\Switch                        scan another folder (same as --roms D:\Switch)
NXsort "D:\Switch\Game.nsp"             process one specific file
```

By default NXsort only **reads**. Nothing on disk is renamed or moved unless you pass `--rename` or `--folderize`, and even then you are asked first.

## Command-line options

| Option | Effect |
|---|---|
| `--keys <path>` | Path to `prod.keys` (default: next to `NXsort.exe`). |
| `--roms <path>` | Folder to scan (default: `roms` next to `NXsort.exe`). A folder given as a plain argument works too. |
| `--noverify` | Skip hashing the content NCAs. Verification is **on** by default. |
| `--rename` | Offer to rename files that don't match the convention. |
| `--recursive` | Also scan subfolders. Combined with `--rename`, also checks folder names. |
| `--folderize` | Offer to move loose files in the roms folder into per-game folders. |
| `--icon` | Extract one icon per game into `icons/`. Always re-reads the files. |
| `--rescan` | Ignore the saved manifest; re-read and re-verify everything. |
| `--nolog` | Don't write `nxsort.log`. |
| `-h`, `--help` | Show usage. |

Any other argument is treated as a file to process (or, if it is an existing directory, as the roms folder). With no files given, every `*.nsp` and `*.xci` in the roms folder is processed.

`--rename` and `--folderize` ask questions on the console, so run them in an interactive terminal.

---

## How a run works

1. **Setup.** Resolve the roms folder, start a fresh log, load `prod.keys`, load `custom.json`/`titles.json` (only when `--rename`, `--folderize` or `--icon` need them), load the manifest.
2. **Phase 1: scan.** For each file, either reuse its saved info (unchanged files) or open it, read every title's CNMT and NACP, print a report, and verify the content hashes. The manifest is saved immediately afterwards so a scan isn't lost if a later prompt is interrupted.
3. **Phase 2: names.** From the scanned facts (and the current `custom.json`/`titles.json`), work out what each file *should* be called. Always check that the `[TITLEID][vREV]` tags in the filename match reality. With `--rename`, offer to fix things.
4. **Folderize** (if requested). Group loose files and offer to move them.
5. **Save** the manifest again, since renames and moves change paths.

The important design point: the manifest stores only **facts read from the files**. Proposed names and folders are recomputed every run from those facts plus your current `custom.json`, so editing `custom.json` takes effect immediately and nothing goes stale.

---

## How NXsort identifies a file

A single `.nsp` or `.xci` can hold one title or several (a "Super NSP" containing a base game, updates and DLC). Each title inside has a type:

- `Application`: the base game
- `Patch`: an update
- `AddOnContent`: DLC

NXsort reduces the whole file to one **identity**, which drives the filename:

**Primary title** (the one whose ID goes in the filename):
1. The `Application`, if there is one.
2. Otherwise the `Patch` with the highest version.
3. Otherwise the first title (a standalone DLC).

**Base title ID.** An `Application` is its own base. Everything else uses the `ApplicationTitleId` from its CNMT (falling back to its own ID if that is zero). Updates and DLC are tied to their game through this ID.

**Version** (the number in `[vREV]`):
- A base game on its own is always **v0**. Its internal revision says nothing useful about what it needs to run.
- A base game bundled with updates uses the **newest update's version**.
- A lone update uses its own version.
- A standalone DLC uses its own version.

**DLC count.** The number of `AddOnContent` titles in the file. A standalone DLC doesn't count itself, so its count is 0.

**Game name.** The first non-blank NACP name from the highest-version update, otherwise the base game's NACP name. After that, name resolution may replace it (see below).

---

## The naming convention

```text
Name [TITLEID][vREV][DLC].ext
```

- **Name** is the game name (see the resolution order below).
- **TITLEID** is the primary title's 16-digit ID, uppercase hex.
- **vREV** is the version number described above, written as plain decimal after a `v`.
- **[DLC]** appears only when the file bundles DLC: `[DLC]` for one, `[DLC3]` for three, and so on. It goes directly after the version tag.
- **ext** is lowercased (`.nsp` or `.xci`).

| What's in the file | Example filename |
|---|---|
| Base game alone | `Skies of Fury [0100ABCD12340000][v0].nsp` |
| Update alone | `Skies of Fury [0100ABCD12340800][v65536].nsp` |
| Base game + updates | `Skies of Fury [0100ABCD12340000][v131072].nsp` |
| Base game + updates + 3 DLC | `Skies of Fury [0100ABCD12340000][v131072][DLC3].nsp` |
| Standalone DLC | `Skies of Fury (Wing Pack)[0100ABCD12341001][v0].nsp` |

(The IDs above are made up for illustration.) Standalone DLC has its own style, covered [below](#standalone-dlc-naming).

---

## Renaming rules in depth

### 1. Where the name comes from

For base games, updates, bundles and XCI files, the name is chosen in this order:

1. **`custom.json`**, keyed by the **base title ID**. Because updates share their base game's key, an update always follows the base game's custom name.
2. **The newest update's NACP name.** NXsort looks across *every* scanned file and the manifest for the highest-version update of that base title. This matters because an update's NACP can carry a newer name than the base game (for example "Skies of Fury" becoming "Skies of Fury DX"), and all files for the game should then agree.
3. **The file's own NACP name** (update first, then base game).
4. **Nothing.** If no name can be found, NXsort can't propose a rename. You can only add one to `custom.json` or skip.

### 2. How names are cleaned

There are two different cleanups, and the difference is deliberate.

**`custom.json` names are the user's own**, so they get only the minimum needed to be a legal Windows filename:

- Characters Windows rejects (`< > : " / \ | ? *`) and control characters become `_`.
- Leading/trailing whitespace and trailing dots/spaces are removed (Windows refuses those).
- Nothing cosmetic is touched. Colons, dashes and trademark symbols you typed are left alone, apart from the characters above that can't exist on disk.

**NACP and `titles.json` names are cleaned more thoroughly:**

| Input | Result |
|---|---|
| `®` `™` `©` `℠` `■` | Replaced with a space, so `COMBAT™7` becomes `COMBAT 7` |
| A standalone ` w/ ` (surrounded by whitespace) | Becomes `with` (otherwise `/` would turn into `_`) |
| `:` and full-width `：` | Replaced with a space, so `A: B` and `A:B` both become `A B` |
| `"` (double quote) | Dropped |
| Invisible/format, private-use and unassigned characters | Replaced with a space (these are often what shows up as `■`) |
| Other characters invalid in filenames | Replaced with `_` |
| Runs of 2+ whitespace | Collapsed to a single space; ends trimmed |

Base game, update, bundle and folder names **keep dashes exactly as written** in the NACP. Only DLC names have dashes cleaned up (see [Standalone DLC naming](#standalone-dlc-naming)).

### 3. When a file counts as "already correct"

NXsort builds the expected filename:

```text
{name} [{TITLEID}][v{version}]{DLC tag}{.ext}
```

Then:

1. If the current filename matches it **exactly** (case-sensitive), nothing happens.
2. If you have a `custom.json` name for this game, a looser test applies: the **name part** of the current filename (everything before the last `[TITLEID][vREV]` tag, trimmed) must equal your custom name exactly, *and* the title ID, version and DLC tags must be correct. This means harmless spacing differences before the tags (`Name[ID][v0]` vs `Name [ID][v0]`) don't trigger a prompt on every run.
3. Otherwise NXsort shows you the mismatch:

```text
Filename doesn't match the "Name [TITLEID][vREV][DLC].nsp" convention:
  current:  Skies of Fury.nsp
  expected: Skies of Fury [0100ABCD12340000][v0].nsp
  [C] Change filename.  Rename this file.
  [A] Add current filename to custom.json.  Remember your preferred name for future runs.
  [S] Skip this time
Choice [C/A/S]:
```

If a `custom.json` name caused the mismatch, the message says *"Filename doesn't match the name in custom.json (...)"* instead.

If the current and expected names are the same length and differ by something other than case, a `[debug]` line points at the first differing character with its Unicode code point. This catches lookalike or hidden characters that make two names look identical.

### 4. The three choices

**[C] Change filename** renames the file to the expected name.
- A name that differs only by **case** is handled correctly (Windows treats those as the same path), by hopping through a temporary `.nxsort_tmp` name.
- If a different file with the expected name already exists, NXsort refuses to overwrite it and leaves your file alone.
- On success the manifest is updated, so the file isn't rescanned next run.

**[A] Add current filename to custom.json** says "my current name is the one I want." NXsort takes the name part of the current filename (everything before the `[TITLEID][vREV]` tags, or the whole stem if there are no tags) and stores it in `custom.json`.
- The key is the **base title ID** for games, updates and bundles, so every file for that game picks up the name. For standalone DLC the key is the DLC's own title ID.
- The file isn't renamed by this choice. Next run, the name part matches your custom name, so you aren't asked again, as long as the tags are correct.
- NXsort immediately re-resolves the name, so the rest of the run (folder check, `--folderize`, icons) uses it.
- `custom.json` is written sorted by title ID (uppercase), indented, with accented characters kept readable.
- If `custom.json` exists but can't be parsed, NXsort refuses to overwrite it and tells you to fix or remove it.

**[S] Skip** (or just press Enter) does nothing this time.

When there is no name available at all, only **[A]** and **[S]** are offered.

### 5. The tag check (always on)

Even without `--rename`, NXsort compares the filename's tags with the truth:

- It finds the **last** `[16 hex digits][v<digits>]` in the filename (whitespace between the two tags is allowed).
- It compares the **title ID**, the **version**, and the **DLC tag** (an optional `[DLC]` or `[DLCn]` directly after the version tag; absent means 0).
- Any mismatch, or missing tags, produces a `WARNING:` that says exactly what's wrong, for example:
  - `title ID in the name is X but the file's real title ID is Y`
  - `version in the name is v5 but base games use v0`
  - `DLC tag in the name implies 2 DLC but the file contains 3`
- Without `--rename`, the warning suggests running with `--rename`. With `--folderize`, it says the file **won't be sorted**.

A tag warning doesn't count as a failed file (exit code unaffected), but it keeps the file out of `--folderize` until it's fixed.

---

## Standalone DLC naming

A DLC `.nsp` has no Control NCA, so it carries no name. NXsort gets one from your files instead.

This applies only to a `.nsp` containing **exactly one title** which is `AddOnContent`, and only with `--rename`. DLC inside a bundle, and DLC in an XCI, follow the normal rules and are counted in the `[DLC]` tag instead.

### Name sources, in order

1. **`custom.json`** entry keyed by the DLC's **own** title ID (used verbatim, after the minimal legal-filename cleanup).
2. **`titles.json`** entry for the DLC's title ID. The base game's name comes from `custom.json` (by base title ID, verbatim) if present, otherwise from `titles.json`. The two are combined as described next.
3. **Nothing.** Only [A] and [S] are offered. The message says whether `titles.json` wasn't loaded at all or simply had no entry.

`titles.json` is read as `{ "TITLEID": { "name": "..." }, ... }`. Only the `name` field is used, IDs are matched case-insensitively, and trailing commas are tolerated.

### Building "Game (DLC name)"

The goal is `Game (DLC name)` without the game name repeated inside the brackets.

1. Trademark symbols are stripped from the DLC name and spaces collapsed.
2. With no base game name available, the cleaned DLC name alone is used.
3. **Prefix match.** Both names are reduced to lowercase letters and digits. If the DLC name begins with the whole game name, that prefix is cut off. The cut is rejected if it lands in the middle of a word (so game `Star` doesn't eat the start of `Starfall Pack`).
4. **Shared-words fallback.** `titles.json` often shortens names, so if there's no full prefix match but the two names share at least **two leading words** (case-insensitive, apostrophes ignored), those words are cut instead. Example: game `Crash Bandicoot 4: It's About Time` and DLC `Crash Bandicoot 4 - Totally Tubular Skins` share `Crash Bandicoot 4`, leaving `Totally Tubular Skins`.
5. After the cut, leading punctuation and separators (`!`, `?`, `:`, dashes, and so on) are dropped, stopping at the first letter or digit so the DLC's real name is never eaten. `LEGO Party! Pre-Order Pack` becomes `Pre-Order Pack`.
6. The remainder is cleaned with the full DLC cleanup, including dash stripping: a ` - ` between words becomes a space, and a dangling dash at the very start or end (separated by a space) is removed. A dash attached to a word (`Pre-Order`) is kept.
7. If nothing is left, or what's left equals the game name, the game name alone is used. Otherwise the name is `Game (Remainder)`.

### The DLC filename

```text
Game (Remainder)[TITLEID][vREV].nsp     name ends with ")", so no space before the tags
Game Name [TITLEID][vREV].nsp           anything else gets the usual space
```

There is no `[DLC]` tag on a standalone DLC file. The same `[C]/[A]/[S]` prompt and the same "name part matches custom.json" shortcut apply. When the expected name came from `titles.json`, the prompt also shows the raw `titles:` name so you can see what it was derived from.

---

## Folder renaming (`--rename --recursive`)

With both flags, NXsort also checks the **folder** a game lives in, but only when all of these hold:

- the file is a **base game** (`Application`). Updates and DLC live alongside their base game, so their titles shouldn't decide the folder name;
- the file is **not directly in the roms folder**;
- a game name is known.

The expected folder name is just the game's display name (custom name if set, otherwise the cleaned NACP name): **no title ID, no version**. If the current folder name differs (case-sensitive), NXsort asks `Rename the folder now? [y/N]`. Case-only renames go through a temporary name, an existing folder with the target name is never overwritten, and the manifest is updated so every file inside follows without a rescan.

## Folder sorting (`--folderize`)

`--folderize` runs last, after all renaming. It only considers files **sitting directly in the roms folder**; anything already inside a subfolder is left alone. Files that got a filename tag warning are **never sorted**: they're listed at the end with their real title ID and expected version so you can fix them first.

**Pass 1: titles with a name** (base games, and updates that carry a NACP name). These are grouped by base title ID. The folder is named after the group's base game (its display name, so a custom name or newest-update name applies). If a folder with exactly that name already exists **anywhere under the roms folder**, it is reused; otherwise a new folder is created in the roms folder.

**Pass 2: nameless titles** (standalone DLC). They're placed using their base title ID:
1. If the base game is loose in the root and getting a folder this run, the DLC joins that group.
2. Otherwise NXsort looks for the base game's folder: a file processed this run, or any container file under the root whose name contains `[BASETITLEID]`.
3. If the base game can't be found, or is itself loose in the root with no folder, the DLC is listed as **skipped** along with its parent ID.

NXsort then prints the plan and asks per group:

```text
Move 3 file(s) into new "Skies of Fury"? [y/N/a]
```

`y` moves that group, anything else skips it, and `a` accepts this and all remaining groups. A move never overwrites: if the destination already has a file with that name, the file is left in place and a warning is logged. Creating folders and moving files are logged, and the manifest follows each move.

## Icons (`--icon`)

For each game (not DLC), NXsort reads the `icon_<Language>.dat` files from the Control NCA's RomFS, trying languages in NACP slot order so American English wins when present. The icon is chosen per game: the **highest-version update that has one**, otherwise the base game's.

It's saved to `icons/` next to `NXsort.exe` as `Name [BASETITLEID].jpg`, using the custom name when set (or just the title ID if no name is known). One fixed folder means later renames and moves can't strand an icon. Details:

- One icon per game: older icons for the same base title (old name, old custom name, different extension) are deleted.
- Within a run, an older version's icon never overwrites a newer one already saved.
- Icons are JPEGs; if one ever isn't (no `FF D8` header), it's saved as `.dat`.
- `--icon` always re-reads the files, ignoring the manifest cache.

---

## Verification

For every content entry listed in a title's CNMT, NXsort hashes the matching NCA inside the container with SHA-256 and compares it to the recorded hash:

| Result | Meaning |
|---|---|
| `[PASS]` | Hash matches |
| `[FAILED]` | Hash mismatch; both hashes are printed. The file may be corrupt or tampered with |
| `[MISSING]` | The NCA listed in the CNMT isn't in the file |
| `[SKIP]` | A `DeltaFragment` that isn't present. This is normal; full patch dumps don't include them |

A file with any `FAILED` or `MISSING` entry is marked as failed. `--noverify` skips all of this (and is much faster).

Files that can't be read at all (no `.cnmt.nca`, XCI without a Secure partition, corrupt data, wrong keys) are reported and logged, **not** recorded in the manifest, and simply retried next run.

## The manifest (scan cache)

`nxsort.manifest.json` records, for each successfully scanned file: path (relative to the roms folder), size, modified time, container type, verification result, and for each title its type, IDs, versions, NACP name/publisher/languages/product code, and each NCA's size and verification result.

- A file is reused without re-reading when its **size and modified time are unchanged**. A file scanned without verification is *not* considered up to date when the current run wants verification.
- If verification is skipped (`--noverify`) on an unchanged file that was previously verified, the earlier results are kept rather than overwritten.
- Entries for files that no longer exist are dropped at startup.
- Renames and moves update the manifest in place.
- Files outside the roms folder are scanned but not remembered.
- If the manifest is missing, unreadable, or from a different schema version, it's copied to `.bak` and replaced with a fresh scan. It's always rebuildable.
- It's saved atomically (temp file, then swap), so an interrupted run can't leave half a manifest.
- Use `--rescan` to force a full re-read and re-verify. `--icon` also forces re-reading.

A cached file prints a one-line "Unchanged since the last scan" summary instead of the full report.

## The log

`nxsort.log` (next to `NXsort.exe`) is **replaced at the start of every run** and appended to as the run goes. One line per event, tagged with a level:

`INFO`, `PASS`, `FAIL`, `WARN`, `ERROR`, `RENAME`, `FOLDER`, `MOVE`, `CUSTOM`, `ICON`

The first line records the roms folder and the arguments used. Logging never throws: if the log can't be written you get one warning and the run continues. `--nolog` disables it.

## Files NXsort reads and writes

| File | Read | Written | Notes |
|---|---|---|---|
| `prod.keys` | ✔ | | Required |
| `custom.json` | ✔ | ✔ (via `[A]`) | `{ "TITLEID": "Preferred name" }`; refuses to overwrite an unparseable file |
| `titles.json` | ✔ | | Optional; only used for standalone DLC names |
| `nxsort.manifest.json` | ✔ | ✔ | Scan cache; `.bak` kept if it was unusable |
| `nxsort.log` | | ✔ | Fresh each run |
| `icons/` | | ✔ | With `--icon` |
| Your `.nsp`/`.xci` files | ✔ | Renamed/moved only with your approval | Contents are never modified |

## Exit codes

| Code | Meaning |
|---|---|
| `0` | Everything processed with no problems |
| `1` | Setup problem: roms folder not found, no `.nsp`/`.xci` files found, or `prod.keys` not found |
| `2` | One or more files failed verification, couldn't be read, or hit an error |

Filename tag warnings do not affect the exit code.

## Building

```text
dotnet build -c Release
dotnet publish -c Release -r win-x64 --self-contained false
```

LibHac is pinned to **0.15.0** deliberately. Its public API (especially `PartitionFileSystem`) has had breaking changes between versions; for example 0.19.0 required an explicit `.Initialize()` call. If you bump the version and it stops compiling, check `hactoolnet`'s `Program.cs` in the LibHac repository for the calling convention that matches.

## Limitations

- Only `.nsp` and `.xci` are handled (no `.nsz`/`.xcz`).
- Everything depends on `prod.keys` being complete and current. Newer titles can need newer master keys.
- Compressed NACP title blocks (the newer HOS 21.0.0+ format) are supported. The order of language slots 18 to 30 is not independently verified.
- NACP names aren't stripped of trailing dots, which Windows rejects in filenames. `custom.json` names are.
- Interactive prompts need a console that can accept input.
