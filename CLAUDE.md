# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A dependency-free static HTML/CSS/JS booking grid (doctors × time slots) that is **embedded inside a 1C:Enterprise host application** ("Doctra"). It is not a standalone web app: the host owns the data, pushes it in as JSON, and receives user actions back as synthetic DOM events. There is no build step, no package manager, no test suite, and no server-side code — the repo is exactly what ships.

Because the host is 1C, all data-payload field names are Cyrillic (`ИД`, `Врач`, `Продолжительность`, …) and mirror 1C metadata. Keep them verbatim; they are a wire contract, not local naming.

## Running it locally

```sh
python -m http.server 8000    # then open http://localhost:8000
```

A real HTTP server is required — `draw.js` bootstraps itself with `XMLHttpRequest`, which browsers block over `file://`.

At the bottom of `js/draw.js`, two fixture files stand in for the host:

```js
readTextFile("js/init.json",         t => doctra_call("init", t));
readTextFile("js/update_cells.json", t => doctra_call("update_cells", t));
```

`js/*.json` is gitignored, so a fresh clone renders nothing. Recover working fixtures from history:

```sh
git show 1c0d697^:js/init.json         > js/init.json
git show 1c0d697^:js/update_cells.json > js/update_cells.json
```

Those snapshots predate several fields (`ИдентификаторСтатуса`, `ЭтоПервичныйПациент`, `ДляБеременных`, `ЧерныйСписок`, `styles`, `compact`) and use the older `clinic.png`-style booking-type image names rather than today's `slot_type_*.png` — extend them by hand when working on newer features. **In production these fixture reads never fire usefully; the host drives everything through `doctra_call`.** Do not add app logic that depends on the JSON files existing.

## Host ⇄ page contract

**Host → page:** the host calls the global `doctra_call(functionName, jsonString)` with `"init"` or `"update_cells"`.

- `init` — one-time setup: registers status/booking-type/insurance appearance and builds the context menu.
- `update_cells` — every subsequent data push, full or partial.

**Page → host:** `sendEvent` / `sendClicks` stash a payload in the module-level `eventProperties`, attach it to a synthetic `MouseEvent("click")` as `evt.doctra_event`, and `dispatchEvent` it on `window`. The host then calls `getEventProperties()`, which **returns and clears** the buffer. This is a single-slot mailbox: firing two events before the host drains one loses the first. Event names sent: `command` (with `{command_name, cells}`), `click`, `doubleclick` (with `{cells}`).

`cells` entries are element ids with the leading `id` stripped (`element.id.substring(2)`) — DOM ids are prefixed `id` because raw 1C ids can begin with a digit.

A hidden `#clickButton` is `.click()`ed on any body double-click; the host listens for it as a separate wake-up signal.

## Rendering model

Everything is absolutely positioned inside three containers (`#headerBlock`, `#timesBlock`, `#containerBlock`) laid out with `display: table`. **JS never computes pixels** — it emits `calc()` strings against CSS custom properties on `:root` (`--slotWidth`, `--hourHeight`, `--borderSize`, `--leftWidth`, `--topHeight`). Resizing the grid means setting those variables, not touching layout code. `update_cells` may set them directly (`hourHeight`, the `compact` preset, or an arbitrary `styles: [{name, value}]` array).

Slot geometry comes from `ОтступВМинутах` (minutes from grid start) and `Продолжительность` (minutes), converted to `top`/`height` via `--hourHeight`; the column comes from the `index` attribute stashed on the matching header div, looked up by `ИДШапки`.

`redraw` in the `update_cells` payload picks the path:

- `redraw: true` — wipe and rebuild header, times, and all slots. Also the only path that honours `text_only` (renders an error message instead of the grid).
- `redraw: false` — incremental patch: each `data` element is located as `#id<ИД>` and re-rendered in place; missing ids are skipped silently; `Удалять: true` removes the element.

Each slot is **two stacked divs**: `.slot` carries all the visuals, and `.slot-layer` is a transparent full-size hit target holding the click/dblclick/contextmenu listeners. `.slot-layer` also carries a `slot` attribute, which is how the container-level `contextmenu` handler distinguishes a right-click on a slot from one on empty space.

## Data-driven styling

Slot colours and icons are **not** in `style.css`. `init()` synthesises CSS rules at runtime via `addClass` → `createCSSSelector`, one per entry in `statuses`, `bookingTypes`, and `bookingInsurances`:

- `.json-<name>` — background/text colour, or `background-image: url('images/<image>')`
- `.json-<name>-highlighted` — left accent border (statuses only)
- `.json-<name>-firsttime` — first-time-patient background (statuses only)

`drawSlots` then applies `json-<Статус>`, `json-<ТипБрони>`, `json-<Страховка>` and the two variants. The `--slot*Color` variables and `.slot-booked`/`.slot-arrived`/… rules in `style.css` are fallbacks for the `ИдентификаторСтатуса` path, not the primary mechanism. Adding a status or insurance is a host-config change plus an image, never a stylesheet edit.

Two badges are the exception and are **hardcoded**, not host-configurable: `ЧерныйСписок` (`.slot-blacklist` → `images/blacklist.png`) and `ДляБеременных` (`.slot-pregnant` / `.slot-not-pregnant`). Their filenames live in `style.css`, and all three PNGs are exactly 12×12 to match the `.slot-type` box — there is no `background-size`, so a replacement at another size will tile or crop.

They are mutually exclusive, blacklist first: a truthy `ЧерныйСписок` renders the blacklist badge and skips the pregnancy check entirely. Otherwise pregnancy is a **tri-state driven by property presence** — key absent renders nothing, truthy renders `pregnant`, and *any* falsy value (`false`, `null`, `0`, `""`) renders `not-pregnant`. Since 1C serializes an unset Boolean as `false`, the host must **omit the key** to show no badge.

## Context menu

Built entirely from `initData.commands` (`{type: "command"|"separator", name, caption, tooltip, icon}`). Captions are Georgian.

Gotcha: the host sends `icon` as a **`.png`** filename, and `init_commands` rewrites it with `.replace("png", "svg")`. The repo ships SVGs. When adding a command, add `images/<name>.svg` and have the host advertise `<name>.png`. (The naive `replace` also mangles any name containing the substring `png` elsewhere.)

## Conventions and traps

- `js/viselect.cjs.js` is vendored `@viselect/vanilla` 3.0.0 (minified) — never edit it. `js/selection.js` configures it: selectables are `.container > div`, and the `beforestart`/`start` handlers deliberately preserve an existing multi-selection on right-click so the context menu can act on the whole set.
- Script order in `index.html` is load-bearing: viselect → `selection.js` (runs immediately, creates the global `selection`) in `<head>`, then `draw.js` after `</body>` so the DOM exists.
- `draw.js` reads `headerBlock`, `timesBlock`, `containerBlock`, `errorBlock`, `divtableBlock`, `clickButton`, and `contextMenu` as bare globals — these only exist via the browser's named-element `window` bindings. **Renaming an `id` in `index.html` (or the menu div in `init_commands`) breaks the JS silently.**
- Similarly, `divElement`, `divGrid`, `divSlot`, `divSlotDetails`, `jsonData`, `initData` are assigned without declaration and leak to `window`. That is the existing style; match it rather than half-converting a function.
- Formatting: tabs, double quotes, semicolons, ~100-col wrap (Prettier defaults with `useTabs`). No linter is configured.
- Fonts are local TTFs in `fonts/` referenced by `@font-face`; there are no CDN or network dependencies anywhere.
