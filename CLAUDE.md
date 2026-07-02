# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Field Notes is a local-only reference notebook for designers: drop in travel photos, and the app names each one, samples its palette, tags materials, and files it by trip (from EXIF GPS). Everything stays on disk — no account, no cloud upload.

## Running

```bash
node server.js                                 # serves http://localhost:4317
PORT=5000 node server.js                       # different port
FIELD_NOTES_MODEL=claude-sonnet-4-6 node server.js   # pin a model (default: claude-haiku-4-5)
CLAUDE_BIN=/full/path/to/claude node server.js # if `claude` isn't auto-found
CHROME_BIN=/full/path/to/chrome node server.js # for zine PDF export / chromeless open (Chromium/Brave/etc.)
```

There is **no build step, no test suite, no linter, and no install step** — the app runs straight from a clone with `node server.js`. The **server** is zero-dependency (Node standard library only, no `node_modules`); the **frontend** may use vendored, self-hosted libraries (see Dependencies below). "Develop" = edit `server.js` / `index.html` and restart the server. The HTML is served `no-cache`, so a browser refresh picks up frontend edits without a restart; **server-side changes require restarting `node server.js`**.

## Hard constraints (do not break these)

These invariants define the project; preserve them unless explicitly asked otherwise:

- **Server stays zero-dependency.** `server.js` uses only Node built-ins (`http`, `fs`, `path`, `child_process`, `crypto`). Don't add npm packages or a `node_modules`/`package.json` to the server — it must keep running from a bare clone.
- **Dependencies: pragmatic, vendored, self-hosted.** Good libraries that make the app better are welcome — don't reinvent the wheel. The bar: **vendor** them as a self-hosted file under `lib/` (committed, served by the `/lib/` route), **no CDN**, **no build step**, so a clone still runs fully offline with just `node server.js`. (First example: Fabric.js for the board editor.) A full npm/bundler toolchain is a bigger, separate decision — raise it rather than assume it.
- **App code lives in `index.html`.** Our own UI/logic stays in the single `index.html` (vanilla JS/CSS, fonts self-hosted under `fonts/`); `lib/` holds only third-party vendored code, not our code.
- **macOS-only by design.** Image work shells out to built-in `sips` (HEIC→JPEG + downscale) and `mdls` (EXIF fallback). ImageMagick (`magick`/`convert`) is an *optional* enhancement for palette + perceptual hash; code must degrade gracefully when it's absent (see the `MAGICK_BIN === ""` paths).
- **Persistence is plain files.** Metadata in `library.json` (human-readable, hand-editable), full-res originals in `library/`. No database, no localStorage. The data model in `library.json` *is* the UI model — a record's fields render directly.

## Architecture

**Two files do everything:** `server.js` (the entire backend) and `index.html` (the entire frontend). They talk over a small JSON API.

### The add pipeline (`POST /api/add` in server.js)

This is the core flow and the most intricate part. The browser sends a data-URL of the original bytes; the server then, in order:
1. **Dedup** — md5 hash of bytes; rejects byte-identical re-imports (against `library.json`) and concurrent double-fires (the in-memory `inFlight` set, since the browser drop handler can fire twice before either is saved).
2. **Store display original** — HEIC is converted to JPEG via `sips`; everything else keeps its original bytes in `library/`.
3. **Downscale for analysis** — a ≤1024px JPEG copy in `.cache/` is what gets sent to Claude (cheaper/faster); the full-res original is never sent.
4. **Analyze** — `analyzeFile` → `runClaudeCLI` (preferred) or `runAPIFromFile` (fallback). Returns the structured record (name, description, category, mood, colors, materials), run through `normalize()` which clamps category/mood to fixed vocabularies and pads colors to 4.
5. **Real palette** — `extractPalette` (ImageMagick histogram) *overrides* the model's guessed hex colors when available, because sampled pixels beat guessed values.
6. **Perceptual hash** (`perceptualHash`, dHash) for near-duplicate clustering in the UI.
7. **Trip + date** — `detectTripAndDate` reads EXIF (`exifFromJpeg` parses TIFF/APP1 bytes directly — does *not* rely on Spotlight, which won't have indexed a just-written file; `mdlsMeta` is the fallback for PNG/WEBP), then `reverseGeocode` turns GPS into a city name (Nominatim, cached on disk in `.cache/geo.json`, ≤1 req/sec).
8. **Persist** — replaces any record with the same `id` (so retries don't duplicate) and **preserves manual edits** (`folder`/`date`/`event` set by hand are never clobbered by auto-detection).

### Image analysis — two backends

`runClaudeCLI` shells out to the local `claude` CLI in headless mode (`-p ... --output-format json --allowedTools Read`), using the user's Claude Code login — **no API key**. `runAPIFromFile` (raw Anthropic API) is used **only** when the CLI isn't found *and* `ANTHROPIC_API_KEY` is set. The `SYSTEM_PROMPT` constant defines the exact output shape and is tuned for short evocative names + a constrained one-word material vocabulary; changing it changes every record's quality. Binary resolution (`CLAUDE_BIN`, `MAGICK_BIN`) searches a hardcoded `EXTRA_PATH` so the server still works under a bare launchd PATH (macOS login auto-start).

### The zine maker

Selected cards lay out as a printable zine. The imposition is built **client-side in `index.html`**, then POSTed to `/api/zine`, which stashes the HTML in-memory (`zineStore`, last 20) and returns an id. It's served back at a real `/zine/<id>` URL because printing from a real page is reliable across browsers; `/api/open` can launch it in a chromeless Chrome app window. The fold-order math is the one place where "looks right" and "is right" diverge — verify with page numbers.

Five layout templates (`buildPagesZine` / `buildFullBleed` / `buildContact` / `buildMiniZine` / `buildBooklet`), all sharing the same building blocks (`doc`/`bar`/`coverBlock`/`metaRows`). `pages`, `full-bleed`, and `contact sheet` are simple sequential page flows (DOM order == reading order); `mini-zine` (PocketMod 8-panel single-sheet, one cut) and `booklet` (saddle-stitch) have real imposition math — **don't touch it without re-verifying page numbers.** The **auto cover** toggle makes `coverBlock` add a representative hero image (most-colorful reference) + a palette strip aggregated over the selection (`listPalette`/`heroItem`); off = the plain text cover.

Three export paths, all from the same built HTML: **create zine** (`/api/zine` → open/print), **save .html** (`/api/zine-file` → `inlineZine` self-contained download), and **save .pdf** (`/api/zine-pdf` → real PDF via headless Chrome `--print-to-pdf`, `renderPDF`). The PDF path renders a **self-contained `file://` copy** (via `inlineZine`) — *not* the live `/zine` URL — because pointing headless Chrome back at this same server makes it fetch its own subresources and then hang without exiting. And it **polls for the finished PDF** (stable size + trailing `%%EOF`) rather than waiting for Chrome to exit, since headless Chrome reliably writes the file but often won't terminate cleanly. Chrome's `--user-data-dir` must live in a **short path** (temp dir, not the deep `.cache`): its unix-domain `SingletonSocket` overflows macOS's ~104-char socket-path cap otherwise and Chrome hangs. Degrades gracefully: no Chrome (or a render failure) returns `{ok:false, reason}` JSON instead of a PDF, and the client falls back to the browser print flow.

### Boards (collage)

Persistent freeform moodboards. `boards.json` (gitignored, same pattern as `library.json`) stores board records — each a list of tiles `{itemId, x, y, w, rot, z}` with geometry **normalized to board width** (4:3 logical aspect, center origin). CRUD lives at `/api/board(s)`; tiles are healed against the library on read so deleting a photo can't orphan a tile. The editor (a fullscreen overlay in `index.html`) is driven by **Fabric.js** (vendored at `lib/fabric.min.js`, served by `/lib/`): we hydrate Fabric objects from tiles on open and serialize back on change (autosaved, debounced). Export is Fabric's native `toDataURL({format:'jpeg', multiplier})`. A board can be created blank, from a trip (`▦` on a trip), or from the current selection.

### Map view

Pins every GPS-tagged reference where it was shot. Coordinates come from the same EXIF path used for trip names (`detectTripAndDate` now also returns `lat`/`lon`, persisted on each record in `library.json`; `/api/backfill` fills them for records that predate the feature, and the client kicks off a backfill on load when any item lacks `lat`). The view is a fullscreen overlay in `index.html` driven by **Leaflet** (vendored at `lib/leaflet.js` + `lib/leaflet.css`, served by `/lib/`). Pins are photo-thumbnail `L.divIcon`s (no marker image assets to vendor); clicking one opens a popup, and clicking the popup opens that reference's modal. Tiles are fetched from OpenStreetMap **at runtime** — same spirit as the runtime Nominatim geocode — but degrade gracefully: on `tileerror` the neutral stage colour shows through and a "tiles unavailable" note appears, so offline you get pins on a plain field, not a broken map. Records without coordinates simply get no pin.

### Frontend (`index.html`)

Loads `/api/library` into an in-memory `items` array and renders a card grid. Major regions, roughly in file order: filter/sort state + `matches()`, near-duplicate clustering (`hamming`/`dupClusters`), color-family bucketing for the color filter, server-call wrappers (`apiAdd`/`apiEdit`/`apiDelete`), rendering (`render`, `cardEl`, folder/event/trip-strip/filter renderers), and the drop-zone ingest path. Cards request downscaled images via `/library-print/<file>?w=N` (cached per width in `.cache/`), never the full-res original.

## Seeding & gitignored state

A fresh clone has no `library.json`; on first run the server seeds it from `samples/` so the grid isn't empty. `library/`, `library.json`, and `.cache/` are gitignored and auto-created — they hold the user's actual collection and scratch, so don't commit them and don't assume they exist when reading code.

## Working conventions

Defaults so you can act without checking in:

- **Just do it within the constraints above.** Don't ask permission for routine work that respects the invariants (server zero-dep, app code in `index.html`, files-as-database, macOS). Adding a well-chosen vendored library is fine; only stop to ask before a heavier departure (an npm/bundler toolchain, a server-side package) — surface the trade-off and the lighter alternative.
- **Prefer the platform, but don't dogmatically avoid libraries.** Reach first for what's already here (`sips`/`mdls`/ImageMagick/Node std-lib, native browser APIs) — it's often enough and keeps things light. But if a vendored library genuinely makes the app better, use it (per the Dependencies rule above) rather than rebuilding it from scratch.
- **New external-tool calls must degrade, not throw.** Match the existing pattern: resolve the binary through `EXTRA_PATH`, wrap the spawn so a missing tool resolves to a safe fallback (`""`, `[]`, original file) rather than failing the request. ImageMagick and the API key are optional; the app must still run without them.
- **Never clobber user data.** Auto-detected fields yield to manual edits (see the `prev`-preservation logic in `/api/add`). When touching persistence, preserve `folder`/`event`/`date` that a user set by hand, and keep `library.json` valid, pretty-printed JSON.

### Verifying changes (there is no test suite)

Run the thing and observe — don't assert it works. The loop:

```bash
node server.js                                    # restart after ANY server.js edit
curl -s localhost:4317/api/library | python3 -m json.tool | head   # library renders + valid JSON
curl -s "localhost:4317/library-print/<file>?w=400" -o /tmp/t.jpg  # sips downscale path
python3 -m json.tool library.json >/dev/null && echo OK            # data file still valid
```

- **Backend change:** restart, then curl the affected endpoint and confirm the response shape and that `library.json` is unchanged-or-correctly-changed.
- **Frontend change:** just refresh the browser (`index.html` is served `no-cache`) — no restart needed.
- **Add-pipeline change:** the honest test is dropping a real photo through the UI, since `POST /api/add` invokes the model and runs the full sips/EXIF/geocode chain (slow, ~seconds). Don't claim the pipeline works off a code read alone.

### Matching the code style

`server.js` and `index.html` are terse on purpose. Match it: compact helpers, single-letter locals in tight loops, and comments that explain **why** (the non-obvious constraint or failure mode being guarded against), not what the line does. New machine-specific behavior gets an env-var override (cf. `PORT`, `FIELD_NOTES_MODEL`, `CLAUDE_BIN`, `MAGICK_BIN`).

### Privacy (this fork is public)

The `SP00KYCHIEF/field-notes` fork is **public**, so anything that gets pushed — commit messages, PR titles/bodies, code comments, branch names — is world-readable.

- **Refer to the maintainer only as the GitHub handle `SP00KYCHIEF`** in anything publicly visible. Never put a real name, email address, or other personal identifier into commits, PRs, comments, or committed files.
- **Never commit personal data.** The user's actual photos and metadata live in `library/`, `library.json`, and `.cache/`, which are gitignored — keep it that way. Don't `git add -f` anything under those paths, don't relax those `.gitignore` rules, and don't paste real EXIF/location/library contents into commit messages or PR descriptions.
- Keep examples and test fixtures to the bundled `samples/` set, never the user's own library.

### Git & commits

**Only ever work on the fork.** All branches, commits, and PRs target `SP00KYCHIEF/field-notes` — never the upstream `Laurencemdonald/field-notes`. When opening a PR, set the base explicitly to the fork's own `main` (`gh pr create --repo SP00KYCHIEF/field-notes --base main`), because GitHub defaults a fork's PR base to the upstream repo. Do not push to, or open PRs against, upstream.

The history follows a consistent shape — match it rather than committing straight to `main`:

- **One feature branch per change**, kebab-case named after the feature (`dup-filter`, `event-tag`, `batch-import`, `trip-palette-strip`). Land via a PR merged into `main`; don't push commits directly to `main`.
- **Commit subject:** `Feature name — short description`, e.g. `Duplicate filter — recognize near-duplicate / burst shots`. Sentence case, em-dash, no trailing period.
- **Commit body:** lead with *why* the change exists / what problem it solves, then a short bulleted list of what changed (split `server:` vs `client:` when both move), then a **`Verified:`** line stating the concrete behavior you observed (actual numbers/output, not "tests pass" — there are none). Add `Closes #N` when it resolves an issue.
- Keep the existing `Co-Authored-By:` and `Claude-Session:` trailers.
- **Auto-committing is fine** — no need to ask before committing routine work. Small docs-only changes can go straight to `main`; keep the branch-per-feature flow for actual code changes.
