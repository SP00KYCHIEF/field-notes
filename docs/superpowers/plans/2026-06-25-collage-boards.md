# Boards (Collage) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add persistent, named freeform "boards" — surfaces to arrange library references on (drag/resize/rotate) and export as a single JPEG.

**Architecture:** A new `boards.json` file-store with four CRUD endpoints in `server.js` (server stays zero-dependency). The editor is a fullscreen overlay in `index.html` driven by a vendored, self-hosted **Fabric.js v6.9.1** canvas. `boards.json` stays our own human-readable model; Fabric is the engine — we hydrate Fabric objects from tiles on open and serialize back on change. Export uses Fabric's native `toDataURL`.

**Tech Stack:** Node stdlib HTTP (existing), vanilla JS in `index.html`, Fabric.js v6.9.1 (vendored UMD at `lib/fabric.min.js`, served by a new `/lib/` route).

**Spec:** `docs/superpowers/specs/2026-06-24-collage-boards-design.md`

**Testing note:** This repo has no test framework (see `CLAUDE.md` → Verification). "Tests" here = the project's real verification loop: `node --check` on the inline script, small standalone Node harnesses for pure logic, `curl` for endpoints, and explicit browser-observation steps for the canvas editor. Don't invent a test runner.

**Coordinate model (read first):** A board has a fixed logical aspect of **4:3**. Each tile stores its **center** position and width **normalized to the on-screen board width `W`** (`x`,`y`,`w` are fractions of `W`; a 4:3 board spans `y` in `0..0.75`). Tiles use Fabric origin `center` so rotation pivots naturally. Height is never stored — it follows the image's natural aspect via uniform scaling. Normalization makes the same board render identically at any canvas size and at export resolution.

---

## File Structure

- **Create `lib/fabric.min.js`** — vendored Fabric v6.9.1 UMD build (third-party; committed so clones run offline). The only file under `lib/`.
- **Modify `server.js`** — add `.js` to `MIME`; add a `/lib/` static route; add `BOARDS_JSON` + `readBoards`/`writeBoards`/`normBoard`; add four `/api/board(s)` routes.
- **Modify `index.html`** — board state vars; `loadBoards()`; `renderBoards()` (sidebar, mirrors `renderEvents`); a "make a board" header button; the editor overlay markup + CSS; the Fabric editor module (open/close, hydrate, add-references picker, autosave, export); Phase B manipulation wiring; Phase C auto-from-trip + from-selection.
- **Modify `.gitignore`** — add `/boards.json`.
- **Modify `CLAUDE.md`** — add a short "Boards" note to the architecture section (the dependency-rule change is already committed on this branch).

All work happens on the existing `collage-boards` branch.

---

## Phase A — Core

### Task A1: Vendor and serve Fabric.js

**Files:**
- Create: `lib/fabric.min.js`
- Modify: `server.js` (the `MIME` map; the static routes near the `/fonts/` route)

- [ ] **Step 1: Download the vendored library**

```bash
mkdir -p lib
curl -fsSL "https://cdn.jsdelivr.net/npm/fabric@6/dist/index.min.js" -o lib/fabric.min.js
# sanity: ~316KB, UMD, version 6.9.1
test "$(wc -c < lib/fabric.min.js)" -gt 250000 && grep -q '"6.9' lib/fabric.min.js && echo OK
```
Expected: `OK`

- [ ] **Step 2: Add `.js` to the MIME map**

In `server.js`, the `MIME` object currently has no JS entry (nothing was served as a script before). Add it:

```js
const MIME = { ".html": "text/html; charset=utf-8", ".js": "text/javascript; charset=utf-8", ".jpg": "image/jpeg", ".jpeg": "image/jpeg", ".png": "image/png", ".webp": "image/webp", ".woff2": "font/woff2", ".css": "text/css; charset=utf-8" };
```

- [ ] **Step 3: Add the `/lib/` static route**

In the route handler, directly after the existing `/fonts/` block, add a matching one:

```js
    // GET /lib/<file> -> vendored, self-hosted third-party libraries (e.g. Fabric.js)
    if (req.method === "GET" && p.startsWith("/lib/")) {
      const name = path.basename(decodeURIComponent(p.slice("/lib/".length)));
      return serveFile(res, path.join(ROOT, "lib", name));
    }
```

- [ ] **Step 4: Verify it serves**

```bash
node server.js >/tmp/fn.log 2>&1 &  SRV=$!; sleep 1.2
curl -s -o /dev/null -w "%{http_code} %{content_type}\n" localhost:4317/lib/fabric.min.js
kill $SRV
```
Expected: `200 text/javascript; charset=utf-8`

- [ ] **Step 5: Commit**

```bash
git add lib/fabric.min.js server.js
git commit -m "Boards: vendor Fabric.js v6.9.1 and serve it via /lib/

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01URSifxdBDR1oLjBm8rhc9d"
```

---

### Task A2: Board store + CRUD endpoints

**Files:**
- Modify: `server.js` (after the `readLib`/`writeLib` helpers; new routes near the other `/api/` routes)
- Test: `scratchpad/board_test.js` (throwaway Node harness)

- [ ] **Step 1: Add the store + normalizer**

After `writeLib` in `server.js`:

```js
const BOARDS_JSON = path.join(ROOT, "boards.json");
function readBoards() {
  try { return JSON.parse(fs.readFileSync(BOARDS_JSON, "utf8")); } catch (e) { return []; }
}
function writeBoards(arr) { fs.writeFileSync(BOARDS_JSON, JSON.stringify(arr, null, 2)); }

// Coerce/clamp an incoming board into our stored shape. Drops unknown fields and
// any tile without an itemId. x/y/w are normalized fractions of board width.
function normBoard(b, prev) {
  const num = (v, d) => { const n = parseFloat(v); return Number.isFinite(n) ? n : d; };
  const clamp = (n, lo, hi) => Math.min(hi, Math.max(lo, n));
  const tiles = (Array.isArray(b.tiles) ? b.tiles : []).map(t => ({
    itemId: String(t.itemId || ""),
    x: clamp(num(t.x, 0.5), -1, 2),
    y: clamp(num(t.y, 0.4), -1, 2),
    w: clamp(num(t.w, 0.25), 0.02, 2),
    rot: num(t.rot, 0),
    z: Math.max(0, Math.floor(num(t.z, 0))),
  })).filter(t => t.itemId);
  const id = (b.id && /^[\w-]+$/.test(b.id)) ? b.id : ("bd_" + Date.now());
  const now = new Date().toISOString();
  return {
    id,
    name: (b.name || "Untitled board").toString().trim().slice(0, 120) || "Untitled board",
    bg: /^#[0-9A-Fa-f]{6}$/.test(b.bg || "") ? b.bg : "#FFFFFF",
    tiles,
    created: (prev && prev.created) || now,
    updated: now,
  };
}
// Drop tiles whose photo no longer exists, so boards self-heal after a delete.
function healBoard(board, libIds) {
  return Object.assign({}, board, { tiles: (board.tiles || []).filter(t => libIds.has(t.itemId)) });
}
```

- [ ] **Step 2: Add the four routes**

Among the other `/api/` routes (e.g. after `GET /api/library`):

```js
    // GET /api/boards -> all boards (tiles healed against the current library)
    if (req.method === "GET" && p === "/api/boards") {
      const libIds = new Set(readLib().map(x => x.id));
      return json(res, 200, readBoards().map(b => healBoard(b, libIds)));
    }

    // GET /api/board?id=... -> one board (healed), 404 if missing
    if (req.method === "GET" && p === "/api/board") {
      const id = url.searchParams.get("id");
      const b = readBoards().find(x => x.id === id);
      if (!b) return json(res, 404, { error: "not found" });
      const libIds = new Set(readLib().map(x => x.id));
      return json(res, 200, healBoard(b, libIds));
    }

    // POST /api/board -> upsert a board (creates id if absent), returns the stored record
    if (req.method === "POST" && p === "/api/board") {
      const b = await body(req);
      let boards = readBoards();
      const prev = b.id ? boards.find(x => x.id === b.id) : null;
      const rec = normBoard(b, prev);
      boards = boards.filter(x => x.id !== rec.id);
      boards.unshift(rec);
      writeBoards(boards);
      return json(res, 200, rec);
    }

    // DELETE /api/board?id=... -> remove a board
    if (req.method === "DELETE" && p === "/api/board") {
      const id = url.searchParams.get("id");
      const boards = readBoards().filter(x => x.id !== id);
      writeBoards(boards);
      return json(res, 200, { ok: true });
    }
```

- [ ] **Step 3: Unit-check the normalizer (pure logic) in Node**

Create `scratchpad/board_test.js`:

```js
// extract normBoard from server.js and exercise it
const fs = require("fs");
const src = fs.readFileSync("server.js", "utf8");
const grab = name => { const i = src.indexOf("function " + name + "("); let d = 0, j = src.indexOf("{", i); for (let k = j; k < src.length; k++){ if(src[k]==="{")d++; else if(src[k]==="}"){d--; if(!d) return src.slice(i,k+1);} } };
eval(grab("normBoard"));
const out = normBoard({ name: "  Trip  ", bg: "nope", tiles: [
  { itemId: "a", x: 0.2, y: 0.3, w: 0.25, rot: -5, z: 2 },
  { itemId: "", x: 0.5 },                 // dropped (no itemId)
  { itemId: "b", w: 9 },                  // w clamped to 2, defaults filled
] });
console.log(JSON.stringify(out, null, 2));
console.assert(out.tiles.length === 2, "should drop the itemId-less tile");
console.assert(out.bg === "#FFFFFF", "bad bg -> default");
console.assert(out.tiles[1].w === 2, "w clamps to 2");
console.assert(out.name === "Trip", "name trimmed");
console.assert(out.id.startsWith("bd_"), "id generated");
console.log("normBoard OK");
```

Run: `node scratchpad/board_test.js`
Expected: prints the object then `normBoard OK` with no assertion errors.

- [ ] **Step 4: Round-trip the endpoints with curl**

```bash
node server.js >/tmp/fn.log 2>&1 &  SRV=$!; sleep 1.2
ID=$(curl -s -XPOST localhost:4317/api/board -H 'content-type: application/json' \
  -d '{"name":"Test","tiles":[{"itemId":"DIFF","x":0.3,"y":0.3,"w":0.3,"rot":0,"z":0},{"itemId":"GHOST","x":0.6,"y":0.6,"w":0.3,"rot":0,"z":1}]}' \
  | python3 -c 'import sys,json;print(json.load(sys.stdin)["id"])')
echo "created $ID"
echo "GET one (GHOST tile should be healed out, leaving 1 tile):"
curl -s "localhost:4317/api/board?id=$ID" | python3 -c 'import sys,json;b=json.load(sys.stdin);print("tiles:",len(b["tiles"]))'
echo "list count:"; curl -s localhost:4317/api/boards | python3 -c 'import sys,json;print(len(json.load(sys.stdin)))'
curl -s -XDELETE "localhost:4317/api/board?id=$ID" >/dev/null
python3 -m json.tool boards.json >/dev/null && echo "boards.json valid"
kill $SRV
```
Expected: `created bd_…`, `tiles: 1` (GHOST not in library → healed out), a list count ≥1, `boards.json valid`.

- [ ] **Step 5: Commit**

```bash
git add server.js
git commit -m "Boards: boards.json store + CRUD endpoints with self-healing tiles

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01URSifxdBDR1oLjBm8rhc9d"
```

---

### Task A3: Ignore the runtime board file

**Files:** Modify: `.gitignore`

- [ ] **Step 1: Add the entry** next to the `/library.json` line:

```
/boards.json
```

- [ ] **Step 2: Verify + commit**

```bash
git check-ignore boards.json && echo ignored
git add .gitignore && git commit -m "Boards: gitignore runtime boards.json

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01URSifxdBDR1oLjBm8rhc9d"
```
Expected: `ignored`.

---

### Task A4: Load Fabric + board state + sidebar list + open/close shell

**Files:** Modify: `index.html` (the `<head>` for the script tag + CSS; the body for editor markup; the `<script>` for state, `loadBoards`, `renderBoards`, editor open/close)

- [ ] **Step 1: Load the library + add the header button**

In `<head>`, just before the app's own `<script>`-less assets (anywhere in `<head>` is fine, but load Fabric before the inline script runs — put it at the end of `<body>` right before `<script>` at line ~242):

```html
<script src="/lib/fabric.min.js"></script>
```

In the header (`header.brand`, after the `#zineStart` button at line ~178):

```html
    <button class="zine-start" id="boardStart">make a board →</button>
```

- [ ] **Step 2: Add the sidebar container + editor overlay markup**

After the `#events` div (line ~193):

```html
  <div class="folders" id="boards"></div>
```

After the item modal overlay (after line ~240), add the board editor overlay:

```html
<div class="board-overlay" id="boardOverlay">
  <div class="board-bar">
    <input class="board-name" id="boardName" placeholder="board name…" autocomplete="off" spellcheck="false">
    <span class="board-link" id="boardAdd">add references</span>
    <span class="board-swatches" id="boardBg"></span>
    <span class="board-link" id="boardExport">export image →</span>
    <span class="board-link" id="boardClose">close</span>
  </div>
  <div class="board-stage" id="boardStage">
    <canvas id="boardCanvas"></canvas>
  </div>
</div>
```

- [ ] **Step 3: Add CSS** (in `<style>`, near the overlay/modal rules):

```css
  .board-overlay{display:none;position:fixed;inset:0;background:var(--bg);z-index:50;flex-direction:column;}
  .board-overlay.open{display:flex;}
  .board-bar{display:flex;align-items:center;gap:22px;padding:16px 24px;border-bottom:1px solid var(--line);}
  .board-name{font-family:var(--serif);font-size:20px;border:none;outline:none;background:transparent;color:var(--ink);min-width:200px;flex:0 1 auto;}
  .board-link{font-family:var(--sans);font-size:11.5px;text-transform:lowercase;letter-spacing:.04em;color:var(--gray);cursor:pointer;}
  .board-link:hover{color:var(--ink);}
  .board-swatches{display:inline-flex;gap:6px;margin-left:auto;}
  .board-swatches .bsw{width:16px;height:16px;border:1px solid var(--line);cursor:pointer;}
  .board-swatches .bsw.active{outline:2px solid var(--ink);outline-offset:1px;}
  .board-stage{flex:1;display:flex;align-items:center;justify-content:center;overflow:hidden;background:#F2F2F0;}
```

- [ ] **Step 4: Add board state + load** (in the inline `<script>`, near the other top-level state like `let items`):

```js
let boards = [];          // loaded board records
let editing = null;       // the board currently open in the editor, or null
let fcanvas = null;       // the Fabric.Canvas instance while editing

async function loadBoards(){
  try { boards = await (await fetch("/api/boards")).json(); }
  catch(e){ boards = []; }
  renderBoards();
}
async function apiBoardSave(b){
  const r = await fetch("/api/board",{method:"POST",headers:{"content-type":"application/json"},body:JSON.stringify(b)});
  return r.json();
}
async function apiBoardDelete(id){ await fetch("/api/board?id="+encodeURIComponent(id),{method:"DELETE"}); }
```

Call `loadBoards()` wherever the app first fetches `/api/library` on startup (alongside the initial render).

- [ ] **Step 5: Render the sidebar boards list** (mirrors `renderEvents`):

```js
function renderBoards(){
  const host=document.getElementById("boards");
  host.innerHTML="";
  const mk=(label,onClick,active)=>{ const el=document.createElement("div"); el.className="fold"+(active?" active":""); el.textContent=label; el.addEventListener("click",onClick); return el; };
  host.appendChild(mk("+ new board",()=>openBoard(null)));
  boards.forEach(b=> host.appendChild(mk(b.name, ()=>openBoard(b.id))));
}
```

- [ ] **Step 6: Open/close the editor shell** (empty canvas, sized to fit, 4:3):

```js
// Size the canvas to the largest 4:3 box that fits the stage, then (re)place tiles.
function fitBoardCanvas(){
  if(!fcanvas) return;
  const stage=document.getElementById("boardStage");
  const pad=32, availW=stage.clientWidth-pad, availH=stage.clientHeight-pad;
  let W=availW, H=W*0.75;
  if(H>availH){ H=availH; W=H/0.75; }
  fcanvas.setDimensions({width:Math.round(W),height:Math.round(H)});
  layoutTiles();                       // reposition every object from normalized coords
  fcanvas.renderAll();
}

async function openBoard(id){
  let board = id ? boards.find(b=>b.id===id) : null;
  if(!board){ board = await apiBoardSave({name:"Untitled board",bg:"#FFFFFF",tiles:[]}); boards.unshift(board); renderBoards(); }
  editing = board;
  document.getElementById("boardOverlay").classList.add("open");
  document.getElementById("boardName").value = board.name;
  fcanvas = new fabric.Canvas("boardCanvas",{ backgroundColor: board.bg||"#FFFFFF", preserveObjectStacking:true });
  fitBoardCanvas();
  await hydrateTiles(board);           // defined in Task A5
  fcanvas.renderAll();
  window.addEventListener("resize", fitBoardCanvas);
}

function closeBoard(){
  window.removeEventListener("resize", fitBoardCanvas);
  if(fcanvas){ fcanvas.dispose(); fcanvas=null; }
  editing=null;
  document.getElementById("boardOverlay").classList.remove("open");
  renderBoards();
}
```

- [ ] **Step 7: Wire the buttons** (near other `addEventListener` setup at the bottom of the script):

```js
document.getElementById("boardStart").addEventListener("click",()=>openBoard(null));
document.getElementById("boardClose").addEventListener("click",closeBoard);
```

Add temporary stubs so the file parses before Task A5 fills them in:

```js
function layoutTiles(){}                 // replaced in Task A5
async function hydrateTiles(){}          // replaced in Task A5
```

- [ ] **Step 8: Syntax check + browser-verify the shell**

```bash
sed -n '243,$p' index.html | sed '/<\/script>/,$d' > /tmp/fn_check.js && node --check /tmp/fn_check.js && echo "JS OK"
```
Then in the running app: reload, confirm a **boards** row shows `+ new board`; click it → fullscreen editor opens with an empty 4:3 canvas; type a name; click **close** → returns to the grid and the new board now appears in the sidebar. Confirm in console `fabric.version === "6.9.1"`.

- [ ] **Step 9: Commit**

```bash
git add index.html
git commit -m "Boards: load Fabric, sidebar boards list, open/close editor shell

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01URSifxdBDR1oLjBm8rhc9d"
```

---

### Task A5: Hydrate tiles, add references, drag, autosave

**Files:** Modify: `index.html` (replace the Task A4 stubs; add the picker; add serialize/autosave)

- [ ] **Step 1: Tile ↔ Fabric mapping + layout/hydrate**

Replace the stubs from Task A4 with:

```js
// Build the /library-print URL for a tile's photo, mid-size for the editor.
function tileImgURL(itemId){ const it=items.find(x=>x.id===itemId); return it&&it.file ? ("/library-print/"+it.file+"?w=900") : null; }

// Place a Fabric image object from a normalized tile, using the current canvas width W.
function applyTile(img, t){
  const W=fcanvas.getWidth();
  const targetW=t.w*W;
  const scale=targetW/(img.width||targetW);   // uniform scale preserves the image's aspect
  img.set({ originX:"center", originY:"center", left:t.x*W, top:t.y*W, scaleX:scale, scaleY:scale, angle:t.rot||0 });
  img.itemId=t.itemId;
  img.setCoords();
}

// Reposition existing objects after a resize (coords are normalized, so re-derive px).
function layoutTiles(){
  if(!fcanvas) return;
  const W=fcanvas.getWidth();
  fcanvas.getObjects().forEach(img=>{
    const t=img._tile; if(!t) return;
    const scale=(t.w*W)/(img.width||1);
    img.set({ left:t.x*W, top:t.y*W, scaleX:scale, scaleY:scale, angle:t.rot||0 });
    img.setCoords();
  });
}

// Load every tile of a board into the canvas (in z order).
async function hydrateTiles(board){
  const ordered=(board.tiles||[]).slice().sort((a,b)=>a.z-b.z);
  for(const t of ordered){
    const u=tileImgURL(t.itemId); if(!u) continue;
    const img=await fabric.FabricImage.fromURL(u,{crossOrigin:"anonymous"});
    img._tile=t;                         // keep the normalized record on the object
    applyTile(img,t);
    fcanvas.add(img);
  }
}

// Read the current canvas back into our normalized tile model.
function serializeTiles(){
  const W=fcanvas.getWidth();
  return fcanvas.getObjects().map((img,i)=>({
    itemId: img.itemId,
    x: img.left/W,
    y: img.top/W,
    w: (img.width*img.scaleX)/W,
    rot: img.angle||0,
    z: i,
  })).filter(t=>t.itemId);
}
```

- [ ] **Step 2: Autosave (debounced) on any change**

```js
let saveTimer=null;
function scheduleSave(){
  if(!editing) return;
  clearTimeout(saveTimer);
  saveTimer=setTimeout(async ()=>{
    editing.name=document.getElementById("boardName").value.trim()||"Untitled board";
    editing.bg=fcanvas.backgroundColor||"#FFFFFF";
    editing.tiles=serializeTiles();
    const saved=await apiBoardSave(editing);
    editing.id=saved.id;
    const i=boards.findIndex(b=>b.id===saved.id); if(i>=0) boards[i]=saved;
    renderBoards();
  },600);
}
```

Hook it up inside `openBoard`, right after `new fabric.Canvas(...)`:

```js
  ["object:modified","object:added","object:removed"].forEach(ev=>fcanvas.on(ev,scheduleSave));
  document.getElementById("boardName").addEventListener("input",scheduleSave);
```

(When hydrating, `object:added` fires per tile — that's fine; it just debounces into one no-op-equivalent save. To avoid a save storm on open, set a flag: `fcanvas._hydrating=true` before `hydrateTiles` and clear it after, and early-return from `scheduleSave` while it's set.)

- [ ] **Step 3: Add-references picker**

Reuse the library grid as a chooser. Minimal approach — a lightweight overlay listing ready items; clicking one drops a tile near center:

```js
function addTile(itemId){
  const u=tileImgURL(itemId); if(!u) return;
  fabric.FabricImage.fromURL(u,{crossOrigin:"anonymous"}).then(img=>{
    const t={itemId, x:0.5+(Math.random()*0.1-0.05), y:0.4+(Math.random()*0.1-0.05), w:0.28, rot:0, z:fcanvas.getObjects().length};
    img._tile=t; applyTile(img,t); fcanvas.add(img); fcanvas.setActiveObject(img); fcanvas.renderAll();
  });
}
function openPicker(){
  // simple prompt-free picker: a column of thumbnails over the stage
  const host=document.getElementById("boardStage");
  let pick=document.getElementById("boardPicker");
  if(pick){ pick.remove(); return; }
  pick=document.createElement("div"); pick.id="boardPicker"; pick.className="board-picker";
  items.filter(it=>it.status==="ready").forEach(it=>{
    const im=document.createElement("img"); im.src="/library-print/"+it.file+"?w=200"; im.title=it.name||"";
    im.addEventListener("click",()=>{ addTile(it.id); });
    pick.appendChild(im);
  });
  host.appendChild(pick);
}
document.getElementById("boardAdd").addEventListener("click",openPicker);
```

CSS for the picker:

```css
  .board-picker{position:absolute;top:0;right:0;bottom:0;width:180px;overflow-y:auto;background:var(--bg);border-left:1px solid var(--line);padding:10px;display:flex;flex-direction:column;gap:8px;}
  .board-picker img{width:100%;cursor:pointer;border:1px solid var(--line);}
  .board-picker img:hover{outline:2px solid var(--ink);}
```

Note: `.board-stage` needs `position:relative;` for the picker's absolute positioning — add it to that rule.

- [ ] **Step 4: Update the serialize keys to read `img._tile` consistently**

`serializeTiles` reads live Fabric props (`left/top/scaleX/angle`) — correct, since dragging mutates those. After a move/scale/rotate, also refresh the stored `_tile` so a later `layoutTiles()` (on resize) uses current values. Add a canvas handler:

```js
  fcanvas.on("object:modified", e=>{ const img=e.target; if(!img) return; const W=fcanvas.getWidth();
    img._tile={ itemId:img.itemId, x:img.left/W, y:img.top/W, w:(img.width*img.scaleX)/W, rot:img.angle||0, z:fcanvas.getObjects().indexOf(img) }; });
```

- [ ] **Step 5: Verify in the browser**

Reload. Open a board → click **add references** → click two thumbnails → two images appear near center. Drag them apart. Wait ~1s (autosave). Reload the whole page, reopen the board → the two tiles are still there in roughly the same spots. Confirm `node --check` passes (Step 8 of A4 command).

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "Boards: hydrate/serialize tiles, add-references picker, drag, autosave

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01URSifxdBDR1oLjBm8rhc9d"
```

---

### Task A6: Export to JPEG

**Files:** Modify: `index.html`

- [ ] **Step 1: Export handler**

```js
function exportBoard(){
  if(!fcanvas) return;
  fcanvas.discardActiveObject(); fcanvas.renderAll();   // don't bake selection handles into the image
  const W=fcanvas.getWidth();
  const multiplier=Math.max(1, 2400/W);                 // render ~2400px wide
  const dataURL=fcanvas.toDataURL({ format:"jpeg", quality:0.92, multiplier });
  const a=document.createElement("a");
  a.href=dataURL;
  a.download=((editing&&editing.name)||"board").replace(/[^\w-]+/g,"_")+".jpg";
  document.body.appendChild(a); a.click(); a.remove();
}
document.getElementById("boardExport").addEventListener("click",exportBoard);
```

- [ ] **Step 2: Verify**

Open a board with a couple of tiles, arrange them, click **export image →**. A `<name>.jpg` downloads. Open it: the composition (positions, sizes, background) matches the on-screen board, at higher resolution, with no selection handles.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "Boards: export the board to a high-res JPEG

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01URSifxdBDR1oLjBm8rhc9d"
```

---

## Phase B — Manipulation & polish

### Task B1: Resize, rotate, layering, delete

Most of this is native to Fabric; this task confirms it's on and wires delete + bring-to-front.

**Files:** Modify: `index.html`

- [ ] **Step 1: Confirm resize/rotate controls** — Fabric shows corner (scale) + rotation controls by default on a selected object; `preserveObjectStacking:true` (set in A4) keeps z-order on selection. To keep tile aspect locked while resizing, set on each image after load (in `applyTile`): `img.lockUniScaling=false` is the default (corner handles already scale uniformly; side handles scale one axis). To force uniform-only, hide the mid controls:

```js
  img.setControlsVisibility({ mt:false, mb:false, ml:false, mr:false }); // corners + rotate only -> aspect preserved
```
Add that line inside `applyTile`.

- [ ] **Step 2: Bring-to-front on selection + delete**

In `openBoard`, after creating `fcanvas`:

```js
  fcanvas.on("selection:created", e=>{ const o=fcanvas.getActiveObject(); if(o){ fcanvas.bringObjectToFront(o); scheduleSave(); } });
  fcanvas.on("selection:updated", e=>{ const o=fcanvas.getActiveObject(); if(o){ fcanvas.bringObjectToFront(o); scheduleSave(); } });
```

Delete via keyboard (guarded so it doesn't fire while typing in the name field):

```js
document.addEventListener("keydown", e=>{
  if(!editing || !fcanvas) return;
  if(document.activeElement && document.activeElement.tagName==="INPUT") return;
  if(e.key==="Backspace" || e.key==="Delete"){ const o=fcanvas.getActiveObject(); if(o){ fcanvas.remove(o); fcanvas.discardActiveObject(); fcanvas.renderAll(); scheduleSave(); e.preventDefault(); } }
});
```

- [ ] **Step 2b: Re-index z after a stacking change** — `serializeTiles` already assigns `z` from the live object order, so bring-to-front is captured on the next save automatically. No extra code.

- [ ] **Step 3: Verify** — select a tile: corner-resize keeps its aspect (no stretch); rotate handle tips it; selecting a tile that was behind brings it in front; Backspace removes the selected tile; all survive a reload (autosave). `node --check` passes.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "Boards: resize (aspect-locked), rotate, layering on select, delete

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01URSifxdBDR1oLjBm8rhc9d"
```

---

### Task B2: Background swatches

**Files:** Modify: `index.html`

- [ ] **Step 1: Render swatches + set background**

```js
const BOARD_BGS=["#FFFFFF","#F4F1EA","#111111","#D9D9D6"]; // white, paper, black, neutral
function renderBoardBg(){
  const host=document.getElementById("boardBg"); host.innerHTML="";
  BOARD_BGS.forEach(c=>{
    const s=document.createElement("span"); s.className="bsw"+((fcanvas&&fcanvas.backgroundColor===c)?" active":"");
    s.style.background=c;
    s.addEventListener("click",()=>{ fcanvas.backgroundColor=c; fcanvas.renderAll(); renderBoardBg(); scheduleSave(); });
    host.appendChild(s);
  });
}
```

Call `renderBoardBg()` at the end of `openBoard`.

- [ ] **Step 2: Verify** — clicking a swatch changes the board background immediately and the active swatch is outlined; reload preserves it; export reflects the chosen background.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "Boards: background swatches (white/paper/black/neutral)

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01URSifxdBDR1oLjBm8rhc9d"
```

---

## Phase C — Convenience

### Task C1: Make a board from a trip

**Files:** Modify: `index.html` (the `mkFold` in `renderFolders`, plus a layout helper)

- [ ] **Step 1: Loose justified grid layout (pure, testable)**

```js
// Lay out n tiles in a loose centered grid, normalized to board width (4:3 board).
// Returns [{x,y,w}] with slight jitter so it doesn't look mechanical.
function gridLayout(n){
  if(n<=0) return [];
  const cols=Math.ceil(Math.sqrt(n*4/3));
  const rows=Math.ceil(n/cols);
  const cellW=1/(cols+0.5), w=cellW*0.82;
  const out=[];
  for(let i=0;i<n;i++){
    const r=Math.floor(i/cols), c=i%cols;
    const x=(c+0.75)/(cols+0.5);
    const y=(r+0.75)/(cols+0.5);            // same unit scale as x (normalized to width)
    const jx=(((i*37)%10)/10-0.5)*0.02, jy=(((i*53)%10)/10-0.5)*0.02;
    out.push({ x:x+jx, y:y+jy, w });
  }
  return out;
}
```

(Jitter is deterministic — no `Math.random()` — so layouts are reproducible.)

- [ ] **Step 2: Add the trip action**

In `renderFolders`'s `mkFold`, for real trips (`key!==null && key!==UNSORTED`), add a small "→ board" affordance, or simplest: a separate handler. Add inside `mkFold` after the drop handlers:

```js
    if(key!==null && key!==UNSORTED){
      const b=document.createElement("span"); b.className="fold-board"; b.textContent="▦"; b.title="make a board from this trip";
      b.addEventListener("click", async e=>{ e.stopPropagation(); await boardFromTrip(key); });
      el.appendChild(b);
    }
```

```js
async function boardFromTrip(trip){
  const ids=items.filter(it=>it.status==="ready" && (it.folder||"").trim()===trip).map(it=>it.id);
  const layout=gridLayout(ids.length);
  const tiles=ids.map((itemId,i)=>({ itemId, x:layout[i].x, y:layout[i].y, w:layout[i].w, rot:0, z:i }));
  const board=await apiBoardSave({ name:trip, bg:"#FFFFFF", tiles });
  boards.unshift(board); renderBoards();
  await openBoard(board.id);
}
```

CSS:

```css
  .fold-board{margin-left:8px;color:var(--gray);cursor:pointer;}
  .fold-board:hover{color:var(--ink);}
```

- [ ] **Step 3: Test the layout in Node**

```bash
node -e '
'"$(sed -n "/function gridLayout/,/^}/p" index.html)"'
const L=gridLayout(5);
console.log(L);
console.assert(L.length===5,"5 tiles");
console.assert(L.every(t=>t.x>0&&t.x<1&&t.y>0&&t.y<0.95&&t.w>0&&t.w<0.5),"in-bounds");
console.log("gridLayout OK");'
```
Expected: prints 5 positions then `gridLayout OK`.

- [ ] **Step 4: Verify in browser** — click the `▦` on a trip with photos → a new board named after the trip opens, pre-filled with that trip's photos in a loose grid. Rearrange, reload → persists.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Boards: make a board auto-seeded from a trip

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01URSifxdBDR1oLjBm8rhc9d"
```

---

### Task C2: Make a board from the current selection

**Files:** Modify: `index.html` (the zine selection toolbar area)

- [ ] **Step 1: Add a toolbar action**

In the zine toolbar (`#zineToolbar`, line ~202), add next to `create zine`:

```html
  <span class="zt-link" id="ztBoard">make board</span>
```

```js
document.getElementById("ztBoard").addEventListener("click", async ()=>{
  const ids=[...selectedIds];
  if(!ids.length) return;
  const layout=gridLayout(ids.length);
  const tiles=ids.map((itemId,i)=>({ itemId, x:layout[i].x, y:layout[i].y, w:layout[i].w, rot:0, z:i }));
  const board=await apiBoardSave({ name:"Untitled board", bg:"#FFFFFF", tiles });
  boards.unshift(board); renderBoards();
  await openBoard(board.id);
});
```

- [ ] **Step 2: Verify** — enter zine-select mode, select a few cards, click **make board** → a board opens seeded with exactly those cards.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "Boards: make a board from the current card selection

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01URSifxdBDR1oLjBm8rhc9d"
```

---

### Task C3: Document the feature in CLAUDE.md

**Files:** Modify: `CLAUDE.md`

- [ ] **Step 1: Add a short "Boards" paragraph** to the Architecture section (after the zine maker description):

```markdown
### Boards (collage)

Persistent freeform moodboards. `boards.json` (gitignored, same pattern as `library.json`) stores board records — each a list of tiles `{itemId, x, y, w, rot, z}` with geometry **normalized to board width** (4:3 logical aspect, center origin). CRUD lives at `/api/board(s)`; tiles are healed against the library on read so deleting a photo can't orphan a tile. The editor (a fullscreen overlay in `index.html`) is driven by **Fabric.js** (vendored at `lib/fabric.min.js`, served by `/lib/`): we hydrate Fabric objects from tiles on open and serialize back on change (autosaved, debounced). Export is Fabric's native `toDataURL({format:'jpeg', multiplier})`. A board can be created blank, from a trip (`▦` on a trip), or from the current selection.
```

- [ ] **Step 2: Commit**

```bash
git add CLAUDE.md
git commit -m "Boards: document the feature in CLAUDE.md

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01URSifxdBDR1oLjBm8rhc9d"
```

---

## Self-Review

**Spec coverage:**
- Data model / boards.json / normalized coords / self-heal → A2 (+ A3 gitignore). ✓
- Four CRUD endpoints → A2. ✓
- Vendored Fabric + `/lib/` route + `.js` MIME → A1. ✓
- Sidebar boards list + new board → A4. ✓
- Editor: hydrate, add references, drag, autosave → A5; resize/rotate/layer/delete → B1; background → B2. ✓
- JPEG export (multiplier, deselect first) → A6. ✓
- Auto-from-trip → C1; from-selection → C2. ✓
- CLAUDE.md follow-up → C3. ✓

**Placeholder scan:** Task A4 introduces deliberate stubs (`layoutTiles`/`hydrateTiles`) that A5 replaces — explicitly called out, not hidden TODOs. No other placeholders.

**Type/name consistency:** `editing`, `fcanvas`, `boards`, `_tile`, `itemId`, `applyTile`, `layoutTiles`, `hydrateTiles`, `serializeTiles`, `scheduleSave`, `gridLayout`, `apiBoardSave`/`apiBoardDelete` are used consistently across tasks. Tile keys `{itemId,x,y,w,rot,z}` match between client serialize, `normBoard`, and the spec. Fabric v6 calls verified against the 6.9.1 build: `fabric.Canvas`, `fabric.FabricImage.fromURL` (Promise), `bringObjectToFront`, `toDataURL({multiplier})`, `discardActiveObject`, events `object:modified|added|removed`, `selection:created|updated`.

**Risk note:** The one area to watch during execution is the Fabric origin/coordinate mapping (center origin + normalized width) under canvas resize — verify the A5/B1 reload-persistence checks carefully, since a wrong origin shows up as tiles drifting on resize or export.
