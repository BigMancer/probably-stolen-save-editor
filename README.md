# Deep Space Pawnshop — Save Editor

**English** · [简体中文](README.zh-CN.md)

A single-file, fully offline save editor for **Probably Stolen** (Chinese title: 深空当铺), a Unity
space-pawnshop sim. Open the HTML page in a browser, drop in a `.es3` save file, edit values, export,
and overwrite the original save.

No install, no build step, no server, no network access. Everything runs inside the page.

```
存档修改器.html      ← the whole editor (open this)
```

> Game save files are personal data — they are **not** part of this repository (see `.gitignore`).

---

## Features

**常用数值 · Common values** — cash, wild favor, day counter, rent / loan timers, store markups,
shop stats (order, traffic, happiness, water purity), faction power, health, client & inspection
timers, playtime. Labelled inputs on one page.

**🧩 库存 · Inventory** — parses the nested inventory JSON (`mainInvJSON`, `soldInvJSON`):

| | |
|---|---|
| List | item name, `identifier`, 🏷 custom name (`CUSTOM_NAME_TAG`), `unitCount` / `unitValue` / `unitBaseValue`, owning container, type tags |
| **`_values` property editor** | per-item list of its `_keys`; every row shows a Chinese label + the raw tag + **one directly editable input**. The edited field is chosen by tag suffix: `*_INT → valueInt`, `*_FLOAT → valueFloat`, `*_LONG → valueLong`, `*_BOOL → valueBool`, otherwise a non-empty `internalValueString` (custom names, lock IDs, manufacturer…); marker-only tags (`IS_OWNED_TAG`, …) get an enable/disable switch. `▸` expands that tag's full 9-field record. |
| Filter | by name / identifier / custom name / type (`MODULE`, `STORAGE`, 🏷 …) |
| Group | by type, owning container, item name or identifier — groups start **collapsed**, with `全部收起` / `全部展开` |
| Batch | tick rows (or **选中本组** / header check-all), then edit a property on any selected item — the same tag + field is written to every other selected item. Type mismatches are skipped, not corrupted. Toggle `批量同步` off to edit one item only. |

**全部字段 · All fields** — full lazy-loaded tree of the entire save (8,122 fields in the reference
save). Embedded JSON strings (`mainInvJSON`, `soldInvJSON`, …) are expanded as sub-structures, so
search goes *inside* the inventory: searching `弹药` finds `playerStore.mainInvJSON ▸ saveItems[43].name`.
Search results are flat, editable rows.

**改动清单 · Change list** — every pending edit with `old → new`, per-item undo, global revert.

**Auto-restore** — the loaded save plus unexported edits are cached in IndexedDB; reopening the page
puts you back exactly where you left off (`忘记上次` clears the cache).

Dark / light theme, Chinese UI.

---

## Usage

1. Double-click `存档修改器.html` (any Chromium-based browser; tested with Edge and Chrome).
2. Click **选择存档** or drag the `.es3` file onto the page.
3. Edit anything.
4. Click **导出修改后的存档** — a file with the same name is downloaded. Overwrite the original save
   with it.

Default save location (Windows):

```
%USERPROFILE%\AppData\LocalLow\Questing Goose Studio\Probably Stolen\save_1.es3
```

(`save_0.es3`, `SaveFile.es3`, `saves_index.es3` live in the same folder.)

**Back up the save before overwriting, and close the game while you replace it.**

---

## How writes work (and why it is safe)

A `.es3` save is **not** standard JSON. In the reference save:

* 60 dictionary keys are unquoted numbers — `{...,8376:{...},-103:{...},...}`
* 20 escape sequences are invalid JSON — `\“` / `\”`
* the whole inventory is a **JSON string inside JSON** (`mainInvJSON`, 1.19 M characters, 287 items)
* newer game builds write the file with indentation and CRLF line endings

So the editor:

1. parses with its own tolerant parser (handles bare numeric keys, invalid escapes, trailing commas)
   that records **character offsets** for every value;
2. on export, replaces **only the byte ranges of the values you actually changed** — everything else,
   including original escapes like `\/`, indentation and line endings, is copied verbatim;
3. for edits inside an embedded JSON string, splices the change into the outer string literal with
   escape-aware patching, so the rest of the 1.19 M-character string is untouched;
4. re-parses the result and re-reads every changed path before downloading — the status bar reports
   `✔ 外层校验通过 / ✔ 内嵌 JSON 校验通过` (outer + embedded JSON checks) or names what failed.

Verified on the reference save: exporting with no edits reproduces the input **byte for byte**;
editing values inside the inventory changed only the intended characters and left the other 285 items
byte-identical.

---

## Testing

Development used Node + jsdom for UI behaviour and a real headless Edge (via `puppeteer-core`) for
layout/geometry checks. Scripts live outside the repo (in a temp folder); the checks they cover:

* tolerant parser: stats, no-edit round-trip equality, changed-path re-read
* UI: inventory rendering, search inside embedded JSON, group collapse/expand, selection + batch sync
* persistence: edit → reload page → save and edits restored
* layout: header vs. row column geometry (`getBoundingClientRect`) exactly equal

## Project memory

`AGENTS.md` records the save-format facts, architecture constraints and testing recipe for anyone
(human or agent) continuing this project.

## Version

**0.1.0** — first tagged release.

## License

Not specified — personal use. The game and its assets belong to Questing Goose Studio.
