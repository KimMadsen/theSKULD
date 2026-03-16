# theSKULD

### **S**anitizer **K**it for **U**seless **L**itter in **D**Proj

<p align="center">
  <img src="logo.png" alt="theSKULD Logo" width="400">
</p>

A Delphi VCL application for cleaning, editing, and managing `.dproj` files — the XML-based project files that Delphi's IDE has a notorious habit of gunking up.

Named after **Skuld**, the youngest of the three Norns in Norse mythology. Together with her sisters Urð (*"that which became"*) and Verðandi (*"that which is happening"*), Skuld governs *"that which shall be"* — the future, obligation, and what must come to pass. The Norns dwell by the Well of Urð beneath Yggdrasil, where they weave the threads of fate, carve runes into wood, and cast lots to shape destiny. In that spirit, theSKULD weaves order from the tangled threads of your `.dproj` files.

---

## Table of Contents

- [The Problem](#the-problem)
- [Features Overview](#features-overview)
- [Requirements](#requirements)
- [Getting Started](#getting-started)
- [The Tree View](#the-tree-view)
- [Version Info Tab](#version-info-tab)
  - [Understanding VerInfo_Keys](#understanding-verinfokeys)
  - [Duplicate Detection and Coloring](#duplicate-detection-and-coloring)
  - [Editing Values](#editing-values)
  - [Resolving Duplicates Manually](#resolving-duplicates-manually)
  - [Smart Auto-Clean All Duplicates](#smart-auto-clean-all-duplicates)
  - [Key Documentation Panel](#key-documentation-panel)
  - [Macros](#macros)
  - [Reverting to Original](#reverting-to-original)
  - [Adding and Deleting Keys](#adding-and-deleting-keys)
  - [Copying the VerInfo_Keys String](#copying-the-verinfokeys-string)
- [Library Paths Tab](#library-paths-tab)
  - [Viewing Paths](#viewing-paths)
  - [Cleaning Duplicates](#cleaning-duplicates)
  - [Manual Path Management](#manual-path-management)
  - [Applying Changes](#applying-changes)
- [Search & Replace Tab](#search--replace-tab)
  - [Finding Values](#finding-values)
  - [Replacing Values](#replacing-values)
  - [Inserting Into All Configs](#inserting-into-all-configs)
  - [Deleting From All Configs](#deleting-from-all-configs)
  - [Replace Whole Value Mode](#replace-whole-value-mode)
- [Promote / Inherit Tab](#promote--inherit-tab)
  - [How Delphi Config Inheritance Works](#how-delphi-config-inheritance-works)
  - [Finding Promotable Keys](#finding-promotable-keys)
  - [Platform Filtering](#platform-filtering)
  - [Performing the Promotion](#performing-the-promotion)
- [Platform Templates Tab](#platform-templates-tab)
  - [Available Templates](#available-templates)
  - [Previewing Missing Keys](#previewing-missing-keys)
  - [Applying a Template](#applying-a-template)
- [Summary Tab](#summary-tab)
- [Saving](#saving)
- [Architecture Notes](#architecture-notes)
- [Disclaimer](#disclaimer)
- [License](#license)

---

## The Problem

Delphi's IDE manages project settings through `.dproj` files — MSBuild XML files containing `<PropertyGroup>` elements with configuration-specific settings. Over time, these files accumulate serious problems:

**VerInfo_Keys duplication**: The `<VerInfo_Keys>` element stores version information as a semicolon-delimited string of `key=value` pairs. Every time you edit version info in the IDE, Delphi often *appends* new values instead of replacing existing ones. A key like `FileVersion` can end up appearing 8-12 times in a single string, with only the last occurrence being effective. This bloats the file and makes manual editing nearly impossible.

**Library path duplication**: Search paths, namespace lists, and debug source paths accumulate duplicate entries through IDE manipulation, particularly `$(Debugger_DebugSourcePath)` and similar macro references appearing multiple times.

**Accidental configuration pollution**: Settings that belong at a parent config level (e.g., "Debug") sometimes end up in specific platform sub-configs (e.g., "Debug / Win32"), creating maintenance nightmares where you need to update the same value in 8 different places.

**Placeholder text**: Delphi inserts placeholder descriptions like *"The reason for accessing the contacts"* for iOS privacy keys. These must be replaced with real descriptions before App Store submission, but it's easy to miss them buried in the XML.

theSKULD addresses all of these problems.

---

## Features Overview

- **VerInfo_Keys deduplication** with smart scoring (favors most recent versions, real text over placeholders, presence of company/copyright info)
- **Manual duplicate resolution** with per-key occurrence picker showing all values and scores
- **Sorted key display** with color-coded rows (red = active duplicates, green = resolved, orange = placeholder text)
- **Library path cleanup** with duplicate and empty entry detection
- **Global search & replace** across all configurations for both VerInfo keys and library paths
- **Bulk insert/delete** of keys and paths across all configurations
- **Configuration promotion** — move common keys from child configs to parent for clean inheritance
- **Platform templates** — inject standard iOS, macOS, Android, or Windows keys
- **Key documentation** — built-in help for 40+ known keys with descriptions, examples, and placeholder warnings
- **Available macros reference** with one-click copy
- **Revert to original** — undo all changes for any configuration
- **Safe XML writes** — each property is tracked back to its originating XML element; writes always go to the correct `<PropertyGroup>`

---

## Requirements

- Embarcadero Delphi (RAD Studio) — any recent version with VCL support (tested with Delphi 12 Athens)
- Windows (VCL application)
- No third-party components required

---


## Getting Started

1. Launch theSKULD
2. Click **Open .dproj** and select a Delphi project file
3. The tree on the left populates with all configurations found in the file
4. The status bar shows how many configs exist and how many have duplicates
5. Click any configuration in the tree to inspect and edit it

**Always work on a copy of your `.dproj` file, or ensure it is under version control before making changes.**

---

## The Tree View

The left panel shows all logical configurations found in the `.dproj` file, sorted alphabetically. Each entry represents a merged view of all `<PropertyGroup>` elements that belong to that configuration.

Badges appear next to entries that need attention:

- **`[N ver dupes]`** — This configuration has N duplicate VerInfo_Keys entries
- **`[N path dupes]`** — This configuration has N duplicate library path entries
- **`[cleaned]`** — Duplicates were resolved for this configuration

The **"Show only configs with VerInfo duplicates"** checkbox filters the tree to only show configurations that have active VerInfo_Keys duplicates — useful when you just want to clean up and get out.

Selecting any entry in the tree updates all tabs on the right simultaneously.

---

## Version Info Tab

### Understanding VerInfo_Keys

Delphi stores version information in a `<VerInfo_Keys>` XML element as a single semicolon-delimited string:

```
FileVersion=2.4.2.81;ProductVersion=2.4.2;ProductName=MyApp;CompanyName=Acme
```

Different platforms use different keys:

- **Windows**: `FileVersion`, `ProductVersion`, `ProgramID`, `FileDescription`, `ProductName`, `CompanyName`, `LegalCopyright`
- **iOS**: `CFBundleName`, `CFBundleVersion`, `CFBundleShortVersionString`, `NSCameraUsageDescription`, and many more `NS*` privacy keys
- **Android**: `package`, `label`, `versionCode`, `versionName`, `minSdkVersion`, `targetSdkVersion`
- **macOS**: Similar to iOS plus `NSHighResolutionCapable`, `LSApplicationCategoryType`, desktop-specific privacy keys

### Duplicate Detection and Coloring

The grid uses three columns: **Key (sorted)**, **Value (click to edit)**, and **Dupes**.

Row coloring:

| Color | Meaning |
|-------|---------|
| **Red/salmon background** | Active duplicate — this key appears multiple times in the raw string |
| **Green background** | Resolved — duplicates were cleaned, showing the kept value |
| **Orange/peach background** | Placeholder text that needs to be replaced (e.g., "The reason for accessing the contacts") |
| **White** | Normal, no issues |

The Dupes column shows:
- `1` — single occurrence (normal)
- `9x DUPE` — 9 occurrences found (needs resolution)
- `OK` — was duplicated, now resolved

### Editing Values

**Inline editing**: Click on the Value column cell for any key and type directly. The Key and Dupes columns are read-only.

**Edit Value button**: For long values (like iOS privacy descriptions), click the **Edit Value...** button to open a full-size memo editor dialog. Line breaks are automatically stripped on OK since VerInfo values must be single-line.

### Resolving Duplicates Manually

Click the **Resolve Duplicates...** button (visible only when duplicates exist) to open a dialog showing every occurrence of each duplicated key.

Each occurrence shows:
- The actual value
- A **score** indicating relevance (higher = better)
- Tags like `[FIRST]`, `[LAST]`, `[BEST]`

**Scoring heuristics:**
- Non-empty values score higher than empty
- Longer values score higher (more info, capped)
- Higher version numbers score higher
- More recent copyright years score higher
- Real company names and copyright text score higher
- Placeholder text ("The reason for...") is penalized
- Pure macro values like `$(MSBuildProjectName)` are slightly penalized

Buttons at the bottom:
- **Keep Best (smart)** — auto-selects the highest-scored occurrence for each key
- **Select All Last** — picks the last occurrence (Delphi's default behavior)
- **Select All First** — picks the first occurrence
- **Apply Selected** — applies your choices and closes

After resolving, occurrences are trimmed to one, the node is marked as `[cleaned]`, and rows turn green.

### Smart Auto-Clean All Duplicates

The toolbar button **Smart Auto-Clean All Duplicates** runs the smart scoring algorithm across every configuration in one click. It also cleans library path duplicates (see Library Paths section). A summary dialog reports what was cleaned.

### Key Documentation Panel

The right side of the Version Info tab shows documentation for the currently selected key:

- **Category** (iOS Bundle, iOS Privacy, Android Manifest, Windows VersionInfo, etc.)
- **Short description** — one-line explanation
- **Detailed description** — multi-line explanation of what the key does
- **Examples** — real-world example values
- **Placeholder warning** — if the default Delphi value is a placeholder that must be replaced
- **Current value** — the value from the grid

For duplicated keys, the panel also shows all occurrences with their scores, with `>>` marking the resolved value.

Over 40 keys are documented, covering iOS bundle configuration, iOS privacy permission descriptions, macOS privacy keys, Android manifest settings, and Windows version info fields.

### Macros

Below the documentation panel is a list of available macros that can be used in VerInfo values:

| Macro | Description |
|-------|-------------|
| `$(MSBuildProjectName)` | Project name without extension |
| `$(ModuleName)` | Module name |
| `$(PROJECTNAME)` | Project name (uppercase variant) |
| `$(Platform)` | Target platform (e.g., Win32, Android64) |
| `$(Config)` | Build configuration (e.g., Debug, Release) |
| `$(BDS)` | RAD Studio install directory |
| `$(BDSBIN)` | RAD Studio bin directory |

Click **Copy Selected Macro** to copy the macro text to the clipboard for pasting into a value.

### Reverting to Original

The **Revert to Original** button restores all keys and values for the selected configuration to their original state as loaded from the file. This undoes all edits, deduplication, and deletions. A confirmation dialog is shown before reverting.

### Adding and Deleting Keys

- **Add Key** — prompts for a key name and value, adds it to the current configuration
- **Delete Key** — removes the selected key after confirmation

### Copying the VerInfo_Keys String

**Copy VerInfo string** copies the cleaned, sorted `key=value;key=value;...` string to the clipboard — useful for pasting into other tools or comparing configurations.

---

## Library Paths Tab

### Viewing Paths

When you select a configuration in the tree, the Library Paths tab shows:

- The configuration name
- A **Property** dropdown listing only the semicolon-delimited path properties that actually exist in this configuration (e.g., `DCC_Namespace`, `DCC_UnitSearchPath`, `Debugger_DebugSourcePath`)
- A listbox showing each path entry on its own line

Supported properties:
- `DCC_UnitSearchPath` — Unit search paths
- `DCC_ResourcePath` — Resource file paths
- `DCC_ObjSearchPath` — Object file paths
- `DCC_IncludePath` — Include file paths
- `DCC_UsePackage` — Runtime packages
- `DCC_Namespace` — Default namespace prefixes
- `Debugger_DebugSourcePath` — Debug source paths

**Note:** Most configurations don't define path properties directly — they inherit from parent configs via MSBuild's property inheritance. The tab only shows properties that are explicitly defined in the selected configuration. If the Property dropdown is empty, that configuration has no local path overrides.

### Cleaning Duplicates

Duplicate and empty entries are highlighted with a **red/salmon background** and tagged with `[DUPLICATE]` or `(empty entry)` in the listbox. The entry count shows how many duplicates exist, e.g., *"3 entries (1 duplicates/empty - shown in red)"*.

Click **Remove Duplicates & Empty** to strip all duplicate and empty entries in one click. This only modifies the listbox — you must click **Apply Changes** to write back to the XML.

The **Smart Auto-Clean All Duplicates** toolbar button also cleans library path duplicates across all configurations automatically.

### Manual Path Management

- **Remove Selected** — deletes the selected path entry
- **Move Up / Move Down** — reorders path entries

### Applying Changes

Click **Apply Changes** to write the modified path list back to the `.dproj` XML. theSKULD writes to the correct `<PropertyGroup>` element that originally contained the property — it never creates duplicates or writes to the wrong element.

---

## Search & Replace Tab

A global search and replace tool that operates across all configurations at once.

### Mode Selection

The radio group switches between two modes:

- **VerInfo Key Values** — searches/replaces within VerInfo_Keys across all configurations
- **Library Path Entries** — searches/replaces within semicolon-delimited path properties

When switching to Library Path mode, the Key Name Filter field is automatically disabled (not applicable to paths).

### Finding Values

Enter search criteria:
- **Key name filter** (VerInfo mode only) — filter by key name (e.g., `FileVersion` to only search FileVersion values)
- **Search text** — the text to find in values or paths
- **Case sensitive** — whether matching is case-sensitive
- **Partial match** — when checked, finds the text anywhere within a value; when unchecked, requires an exact full match

Click **Find All** to populate the results grid with all matches showing Configuration, Key/Property, Value/Path, and Status.

**Tip:** Double-click any result row to copy its value into the "Replace with" field.

### Replacing Values

Enter the replacement text in **Replace with**, then click **Replace All**.

The **Replace entire value** checkbox controls what happens when a partial match is found:

| Partial Match | Replace Entire Value | Behavior |
|--------------|---------------------|----------|
| ✓ | ✗ | Only the matched substring is replaced. `C:\dev\kbmMW\Source` with search `kbmMW` → replace `kbmMW2` becomes `C:\dev\kbmMW2\Source` |
| ✓ | ✓ | If the search text is found anywhere, the **entire** value is replaced. `C:\dev\kbmMW\Source` with search `kbmMW` → replace `D:\libs\new\Source` becomes `D:\libs\new\Source` |
| ✗ | (ignored) | Only exact full matches are replaced |

A confirmation dialog shows which mode is active before proceeding.

### Inserting Into All Configs

**VerInfo mode:** Enter the key name in "Key name filter" and the value in "Search text". Click **Insert Key Into All** to add that key=value pair to every VerInfo configuration that doesn't already have the key.

**Path mode:** Enter the path text in "Search text". Click **Insert Path Into All** to append that path entry to every configuration that has the specified property.

### Deleting From All Configs

**VerInfo mode:** Enter a key name and/or value to match. Click **Delete Key From All** to remove matching keys from all configurations.

**Path mode:** Enter path text to match. Click **Delete Path From All** to remove matching path entries everywhere.

---

## Promote / Inherit Tab

### How Delphi Config Inheritance Works

Delphi's MSBuild structure uses configuration inheritance. A config like "Debug / Win32" inherits from "Debug", which inherits from "Base / Win32", which inherits from "Base". Each level only needs to define *overrides* — values that differ from the parent.

However, the IDE sometimes writes values into child configs that should be at the parent level. For example, if all your iOS Debug configs have the same `NSCameraUsageDescription`, that value should be in the parent "Debug" config (or even "Base") rather than duplicated across `Debug / iOSDevice64` and `Debug / iOSSimARM64`.

### Finding Promotable Keys

1. Select a **Parent config** from the dropdown (shows configs that have 2+ children with VerInfo_Keys)
2. Optionally set a **Platform filter** to compare only children of the same platform family
3. Click **Find Promotable Keys**

The tool compares all (filtered) children and identifies VerInfo keys that have **identical values across all of them**. These keys are candidates for promotion — they can be moved to the parent and removed from the children, letting inheritance handle it.

A label shows exactly which children are being compared.

### Platform Filtering

The platform filter is important because different platform families have completely different keys. Comparing "Base / Win32" with "Base / iOSDevice64" would find no common keys because Windows uses `FileVersion` while iOS uses `CFBundleVersion`.

Available filters:
- **(All platforms)** — compare all children
- **iOS (iOSDevice64, iOSSimARM64)** — compare only iOS children
- **Android (Android, Android64)** — compare only Android children
- **Windows (Win32, Win64)** — compare only Windows children
- **macOS (OSX64, OSXARM64)** — compare only macOS children

### Performing the Promotion

1. Review the promotable keys in the grid
2. Toggle individual keys with the Promote? column (edit to "Yes" or "No")
3. Use **Select all** checkbox for bulk selection
4. Click **Promote Selected to Parent**

The selected keys are added to the parent configuration and removed from all children. If the parent doesn't have a `<VerInfo_Keys>` element yet, one will be created.

---

## Platform Templates Tab

### Available Templates

theSKULD includes built-in templates for standard platform-specific keys:

| Template | Description |
|----------|-------------|
| iOS Distribution | All required CFBundle and NS* privacy keys for App Store distribution |
| iOS + Common Extras | iOS keys plus FileVersion, ProductVersion, CompanyName fields |
| macOS (OSX64 / OSXARM64) | Standard macOS bundle and privacy keys |
| Android (Android / Android64) | Package, version, SDK, and theme settings |
| Windows (Win32 / Win64) | Standard Windows version info fields |

### Previewing Missing Keys

1. Select a **Target config** from the dropdown
2. Select a **Template**
3. Click **Preview**

The results grid shows each template key with status:
- **EXISTS** — this key is already present in the configuration
- **MISSING** — this key will be added if you apply the template

### Applying a Template

Click **Apply** to add all missing keys with their default values. Existing keys are never modified — only missing ones are inserted.

This is particularly useful for iOS configs where Delphi's defaults often miss required privacy keys, or when setting up a new platform target.

---

## Summary Tab

A read-only text report showing:

- File path, project name, and framework type
- Configuration name mapping (Cfg_1 = Debug, Cfg_2 = Release, etc.)
- Every VerInfo configuration with all its keys and values
- Duplicate and resolved status indicators

Useful for getting a bird's-eye view of the entire project's version info state.

---

## Saving

- **Save** — writes back to the original file
- **Save As...** — writes to a new file

theSKULD writes changes through the XML DOM, preserving the overall structure of the `.dproj` file. Each property is tracked back to the specific `<PropertyGroup>` XML element it came from — a configuration that appears as multiple `<PropertyGroup>` elements in the XML (one for hierarchy definition, another for actual settings) is handled correctly, with writes always going to the right element.

**Important:** Always keep a backup or use version control. While theSKULD is designed to be safe, `.dproj` files are critical to your project.

---

## Architecture Notes

### Composite PropertyGroup Merging

A single logical configuration (e.g., "Base / Win32") may correspond to multiple `<PropertyGroup>` elements in the XML:

1. A hierarchy-defining PG: `<PropertyGroup Condition="('$(Platform)'=='Win32' and '$(Base)'=='true') or '$(Base_Win32)'!=''">` — contains only flags like `Base_Win32=true`, `CfgParent=Base`
2. A settings PG: `<PropertyGroup Condition="'$(Base_Win32)'!=''">` — contains the actual properties like `DCC_Namespace`, `VerInfo_Keys`, `Icon_MainIcon`

theSKULD's parser (`uDprojParser.pas`) merges these into a single `TConfigNode` with multiple `TPropertyGroupRef` entries. Each `TPropertyGroupRef` tracks which `IXMLNode` it came from and which properties it "owns". When writing, `WriteProperty` finds the correct `TPropertyGroupRef` for the property being modified and writes to that specific XML node. New properties are written to the PG with the most existing properties (the "settings" PG).

This design ensures that saves never corrupt the file by writing to the wrong `<PropertyGroup>`.

### Smart Deduplication Scoring

The `ScoreVerInfoValue` function scores each occurrence of a duplicated key:

- +10 base for non-empty
- +0-10 for length (more content = more info)
- -5 for pure macro references like `$(MSBuildProjectName)`
- -8 for placeholder text containing "The reason for"
- Version-aware: higher major version numbers and more dots score higher
- Year-aware: `Copyright 2025` scores higher than `Copyright 2022`
- +8 for real CompanyName (not empty, not a macro)
- +8 for non-empty LegalCopyright

---

## Disclaimer

**USE AT YOUR OWN RISK.**

theSKULD modifies `.dproj` files which are critical to your Delphi projects. While every effort has been made to ensure safe and correct file handling:

- **Always back up your `.dproj` files before using theSKULD**, or ensure they are under version control (Git, SVN, etc.)
- **Test your project thoroughly after making changes** — verify that it compiles, runs, and deploys correctly for all target platforms
- **The authors accept no responsibility** for any damage, data loss, build failures, or other problems resulting from the use of this tool
- `.dproj` files have complex MSBuild semantics including property inheritance, conditional evaluation, and platform-specific overrides — theSKULD edits the XML structure but does not evaluate MSBuild expressions
- Some changes (particularly promotion and template application) alter the inheritance structure of your project settings — review changes carefully before saving

This software is provided "as is", without warranty of any kind, express or implied, including but not limited to the warranties of merchantability, fitness for a particular purpose, and noninfringement. In no event shall the authors be liable for any claim, damages, or other liability arising from the use of this software.

---

## License

MIT License

Copyright (c) 2025

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
