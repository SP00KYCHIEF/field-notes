# Boards — persistent freeform moodboards

**Status:** design approved, pending spec review
**Date:** 2026-06-24
**Scope:** new feature, client + server, within Field Notes' existing constraints

## Summary

A "board" is a persistent, named freeform surface where references from anywhere
in the library are placed, moved, resized, and rotated — a space to *think with*
references — and then exported as a single composed image to post or send.

This is distinct from the zine maker: the zine is print-first and template-driven
(fixed imposition, prints to Letter); a board is screen/share-first, freeform, and
saved for ongoing tweaking.

## Goals

- Arrange references freely on a canvas (drag, resize, rotate, layer) and have the
  arrangement persist across sessions.
- Pull references onto a board from anywhere in the library, not just one trip.
- Create a board auto-seeded from a trip's photos in one click.
- Export the board as a single JPEG image suitable for posting/texting.

## Non-goals (v1)

- Drawing, text annotations, stickers, or shapes — images only.
- Multi-select / group transforms of tiles.
- Sharing/hosting a board as a live URL (export is a downloaded image).
- Collaborative or multi-user editing.

## Decisions (locked)

- **Rotation:** included (per-tile rotate handle — native in Fabric).
- **Saving:** autosave, debounced, after every change. No save button.
- **Export format:** JPEG.
- **Build order:** phased (see Phasing).
- **Editor engine:** **Fabric.js**, vendored as a single self-hosted UMD file in
  `/lib/` (no npm, no build step, works offline). This is a deliberate, scoped
  relaxation of the previously-absolute zero-dependency constraint — see
  "Constraints" and the CLAUDE.md follow-up below. Fabric gives us
  `fabric.Image` objects with built-in move/resize/rotate controls, stacking
  order, and hi-res canvas export, replacing most hand-written interaction code.

## Data model & persistence

New `boards.json` at the repo root, **gitignored** like `library.json` and
`library/` (it references the user's personal photos). Read/written with the same
`readLib`/`writeLib` pattern in `server.js`.

```jsonc
{
  "id": "bd_<timestamp>",
  "name": "Untitled board",
  "bg": "#FFFFFF",            // board background (swatch choice)
  "tiles": [
    {
      "itemId": "<library item id>",
      "x": 0.12,              // left, fraction of board width  (0..1)
      "y": 0.20,              // top,  fraction of board width  (see note)
      "w": 0.30,              // tile width, fraction of board width
      "rot": -4,              // degrees
      "z": 3                  // stacking order (higher = front)
    }
  ],
  "created": "<iso>",
  "updated": "<iso>"
}
```

- The board has a **fixed logical aspect ratio** (4:3). All tile geometry is stored
  **normalized to board width**, so `y` is also expressed in board-width units
  (a 4:3 board spans `y` 0..0.75). Normalized coords mean the on-screen board and
  the high-res export render identically — only the scale factor differs.
- **Tile height is derived**, not stored: `h = w * (naturalHeight / naturalWidth)`
  of the referenced image, so tiles always preserve their image's aspect.
- **Self-healing:** on load, tiles whose `itemId` is no longer in `library.json`
  are dropped silently (handles a photo deleted after being placed).

**Why our own model rather than Fabric's native JSON:** `boards.json` stays the
human-readable, hand-editable, version-independent record the project values (same
as `library.json`). Fabric is the *engine*, not the storage format. On open we
**hydrate** Fabric objects from our tiles; on change we **serialize back** to our
shape — the mapping is direct:

| our tile | Fabric object |
|----------|---------------|
| `x`, `y` | `left/boardW`, `top/boardW` (normalized) |
| `w`      | `(width*scaleX)/boardW` |
| `rot`    | `angle` |
| `z`      | object index in the canvas stack |

`itemId` is stashed on each Fabric object (custom prop) so we can map back.

## Server

Four endpoints, mirroring the existing tiny-router + `readLib`/`writeLib` style.
No other server changes; no new dependencies.

| Method | Path              | Body / params            | Returns                    |
|--------|-------------------|--------------------------|----------------------------|
| GET    | `/api/boards`     | —                        | array of board records     |
| GET    | `/api/board`      | `?id=`                   | one board, 404 if missing  |
| POST   | `/api/board`      | `{id?, name, bg, tiles}` | upserted board (creates id if absent) |
| DELETE | `/api/board`      | `?id=`                   | `{ok:true}`                |

- `POST` validates shape defensively (tiles is an array; numeric fields coerced/
  clamped to sane ranges) and stamps `updated`. Unknown/extra fields are dropped.
- "Auto-generate from trip" needs **no special endpoint**: the client builds the
  initial tile layout from a trip's items and POSTs a normal board.

**Serving the vendored library.** `server.js` gains a `/lib/<file>` static route
(same shape as the existing `/fonts/` route) serving `./lib/`, and `.js` is added
to the `MIME` map (`text/javascript; charset=utf-8` — it currently has no JS entry
because nothing was served as a script before). `lib/fabric.min.js` (Fabric v6 UMD
build, MIT-licensed) is **committed** to the repo, not gitignored — a fresh clone
must run fully offline, which is the whole reason to vendor rather than CDN.

## Client (all in `index.html`)

### Sidebar
- A new **boards** group in the left column, beside trips/events: lists saved
  boards (by name) + a "new board" action. Clicking a board opens the editor.
- Each trip row gains a **"make a board"** action → creates a board auto-seeded
  with that trip's `ready` photos in a loose justified grid, then opens it.
- A board can also be created from the **current card selection** (reusing the
  existing `selecting`/`selectedIds` mechanism).

### Editor (fullscreen overlay)
- A single **`fabric.Canvas`** fills the viewport at the fixed 4:3 aspect, scaling
  to fit. Tiles are `fabric.Image` objects, each carrying a custom `itemId` prop.
- **Interactions (mostly native to Fabric):**
  - Drag to move; corner controls to resize (aspect locked via
    `lockUniScaling`/equal scale); rotate handle for angle.
  - Selecting an object and `bringToFront` on selection gives layering; Backspace
    removes the active object.
  - "add references" opens a picker (the library grid in a chooser) to add tiles;
    new tiles drop near center at a default width with a slight offset.
- **Background:** a small swatch row sets `canvas.backgroundColor`
  (white / paper / black / neutral).
- **Autosave:** Fabric's `object:modified` / `object:added` / `object:removed`
  events mark the board dirty and schedule a debounced (~600ms) serialize →
  `POST /api/board`. Inline-editable name.
- **Export image:** `canvas.toDataURL({ format:'jpeg', quality:0.92, multiplier })`
  where `multiplier` scales the on-screen canvas up to a ~2400px-wide render →
  convert to a Blob → download `<board name>.jpg`. (Deselect any active object
  first so selection handles aren't baked into the image.)
- **Close** returns to the library grid.

### Images
- Tiles load via `/library-print/<file>?w=…` (downscaled, cached) at a mid-size
  width for the editor. For export, Fabric's `multiplier` upscales the existing
  canvas; if that proves soft, a later refinement can swap in higher-res sources
  before the export render. v1 relies on `multiplier`.

## Phasing

- **Phase A (core):** vendor `lib/fabric.min.js` + `/lib/` route + `.js` MIME;
  `boards.json` + the four endpoints; sidebar boards list + new board; Fabric
  editor with add-references + drag + autosave; JPEG export.
- **Phase B (manipulation):** resize, rotate, layering/z-order, background swatches,
  per-tile delete (most are Fabric defaults — this phase is wiring + polish).
- **Phase C (convenience):** "make a board from this trip" auto-layout; create from
  current selection.

Each phase is independently reviewable and leaves the app working.

## Constraints — what holds, what changes

- **Server stays zero-dependency** — `server.js` still uses only Node's standard
  library; the new endpoints and `/lib/` route add no packages.
- **Frontend constraint relaxes (scoped):** the previously-absolute
  "no dependencies, vanilla only" rule now permits **vendored, self-hosted
  client libraries** — no npm, no build step, no CDN, committed to the repo so a
  clone runs offline. Fabric.js is the first such library. **Action: update
  `CLAUDE.md`** to reflect this (server zero-dep; frontend = vanilla + a small set
  of vendored self-hosted libs) so future sessions don't treat the old rule as law.
- **One-file frontend** — all app UI/logic still lives in `index.html`; `/lib/`
  holds only third-party vendored code, not our code.
- **Files-as-database** — `boards.json`, human-readable, gitignored like
  `library.json`. `lib/fabric.min.js` is the one third-party file that IS
  committed (so offline clones work).

## Edge cases

- Empty board: editor opens with just the background; export produces a blank bg
  image (acceptable).
- Photo deleted while on a board: tile dropped on next load (self-heal).
- Board referencing zero surviving items: still listed; opens empty.
- Very large/small tiles: `w` clamped to a sane range (e.g. 0.05–1.5) so a tile
  can bleed off-edge intentionally but not vanish or overflow absurdly.
- Concurrent edits to the same board across two tabs: last write wins (acceptable
  for a single-user local app; matches `library.json` behavior).

## Verification

- Server: curl each endpoint; confirm `boards.json` round-trips and stays valid
  JSON; confirm a tile with a missing `itemId` is dropped on GET.
- Client: create a board, place/drag/resize/rotate tiles, refresh → arrangement
  persists; export → opened JPEG matches the on-screen arrangement (position,
  scale, rotation, background); delete a referenced photo → tile gone on reopen.
- Confirm `library.json`/`library/` are untouched by board ops; confirm the only
  added third-party code is the vendored `lib/fabric.min.js` (server still has zero
  npm dependencies); confirm `/lib/fabric.min.js` is served with a JS content-type
  and the app loads it with no network calls beyond `localhost`.
- Add `boards.json` to `.gitignore`; confirm it is not tracked.
