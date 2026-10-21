---
name: Library/MG/altre_librerie
version: 0.01
versionDate: 2026-10-21
tags: meta/library
---

# Inline table editing
* Click a cell to edit its markdown in place; Enter or clicking away saves, Esc cancels
* Tab / Shift+Tab save and move to the next / previous cell; Tab on the last cell adds a row
* Alt+click (or Ctrl+click) still opens the table source; links and checkboxes in cells work as before
* The ⋮ handle on a hovered cell inserts, moves and deletes rows and columns, and sets column alignment
* Drag the grip left of a row, or on top of a column header, to reorder; Esc cancels the drag
* Every change is one ordinary edit, so Ctrl+Z undoes it. Tables whose pipes are already aligned stay aligned; compact tables stay compact
* Only tables written in the page are editable; query and Lua output is left alone

## Implementazione


```space-style
/* Tables fill the editor width and wrap text instead of scrolling */
.sb-table-widget {
  overflow-x: visible !important;
}
.sb-table-widget table {
  width: 100% !important;
  table-layout: auto;
}
.sb-table-widget td,
.sb-table-widget th {
  white-space: normal !important;
  overflow-wrap: anywhere;
  word-break: break-word;
}
```
```space-style
/* Column resizing (script below): border-box keeps fixed-layout widths summing to 100% */
.sb-table-widget td,
.sb-table-widget th {
  box-sizing: border-box;
}
body.sb-col-resize-hover .sb-table-widget,
body.sb-col-resize-hover .sb-table-widget *,
body.sb-col-resizing,
body.sb-col-resizing * {
  cursor: col-resize !important;
}
body.sb-col-resizing {
  user-select: none;
}
```

# Resizable columns
* Drag a header cell's border to resize the two columns either side of it
* Double-click a header border to reset the table
* Widths are kept per page + header row in this browser's localStorage

```space-lua
local doc = js.window.document
if not doc.getElementById("sb-table-resize") then
  local s = doc.createElement("script")
  s.id = "sb-table-resize"
  s.innerHTML = [[
(function () {
  const KEY = "sb-table-widths", EDGE = 6, MIN = 40;
  let saved = {};
  try { saved = JSON.parse(localStorage.getItem(KEY)) || {}; } catch (e) {}
  const persist = () => { try { localStorage.setItem(KEY, JSON.stringify(saved)); } catch (e) {} };

  const headCells = t => (t.rows.length ? Array.from(t.rows[0].cells) : []);
  const keyOf = t => decodeURIComponent(location.pathname) + "|" +
    headCells(t).map(c => c.textContent.trim()).join("|");

  function apply(t, pcts) {
    t.style.tableLayout = "fixed";
    headCells(t).forEach((c, i) => { c.style.width = pcts[i] + "%"; });
  }
  function reset(t) {
    t.style.tableLayout = "";
    headCells(t).forEach(c => { c.style.width = ""; });
  }
  function restore() {
    document.querySelectorAll(".sb-table-widget table").forEach(t => {
      const p = saved[keyOf(t)];
      if (p && p.length === headCells(t).length) apply(t, p);
    });
  }

  // { t, i } when the pointer is on the border between header cells i and i+1
  function edgeAt(e) {
    const cell = e.target.closest && e.target.closest(".sb-table-widget th, .sb-table-widget td");
    if (!cell) return null;
    const t = cell.closest("table");
    if (!t || cell.parentElement !== t.rows[0]) return null;
    const cells = headCells(t), idx = cells.indexOf(cell), r = cell.getBoundingClientRect();
    if (r.right - e.clientX <= EDGE && idx < cells.length - 1) return { t, i: idx };
    if (e.clientX - r.left <= EDGE && idx > 0) return { t, i: idx - 1 };
    return null;
  }

  let dragging = false, swallowClick = false;

  document.addEventListener("mousemove", e => {
    if (!dragging) document.body.classList.toggle("sb-col-resize-hover", !!edgeAt(e));
  }, { passive: true });

  // Keep CodeMirror from moving the cursor into the table source
  const block = e => {
    if (dragging || swallowClick || edgeAt(e)) { e.preventDefault(); e.stopPropagation(); }
  };
  document.addEventListener("mousedown", block, true);
  document.addEventListener("click", block, true);

  document.addEventListener("pointerdown", e => {
    if (e.button !== 0) return;
    const hit = edgeAt(e);
    if (!hit) return;
    e.preventDefault(); e.stopPropagation();
    const { t, i } = hit;
    const total = t.getBoundingClientRect().width;
    const px = headCells(t).map(c => c.getBoundingClientRect().width);
    const pct = () => px.map(w => +(w / total * 100).toFixed(2));
    const x0 = e.clientX, a = px[i], b = px[i + 1];
    apply(t, pct());
    dragging = true;
    document.body.classList.add("sb-col-resizing");
    const move = ev => {
      const d = Math.max(MIN - a, Math.min(b - MIN, ev.clientX - x0));
      px[i] = a + d; px[i + 1] = b - d;
      apply(t, pct());
    };
    const up = () => {
      window.removeEventListener("pointermove", move);
      dragging = false;
      document.body.classList.remove("sb-col-resizing");
      saved[keyOf(t)] = pct();
      persist();
      swallowClick = true;
      setTimeout(() => { swallowClick = false; }, 0);
    };
    window.addEventListener("pointermove", move);
    window.addEventListener("pointerup", up, { once: true });
  }, true);

  document.addEventListener("dblclick", e => {
    const hit = edgeAt(e);
    if (!hit) return;
    e.preventDefault(); e.stopPropagation();
    delete saved[keyOf(hit.t)];
    persist();
    reset(hit.t);
  }, true);

  // CodeMirror rebuilds table widgets on edit/scroll; re-apply saved widths
  let queued = false;
  new MutationObserver(() => {
    if (queued) return;
    queued = true;
    requestAnimationFrame(() => { queued = false; restore(); });
  }).observe(document.body, { childList: true, subtree: true });
  restore();
})();
]]
  doc.head.appendChild(s)
end
```

# Inline editing
* Click a cell to edit its markdown in place; Enter or clicking away saves, Esc cancels
* Tab / Shift+Tab save and move to the next / previous cell; Tab on the last cell adds a row
* Alt+click (or Ctrl+click) still opens the table source; links and checkboxes in cells work as before
* The ⋮ handle on a hovered cell inserts, moves and deletes rows and columns, and sets column alignment
* Drag the grip left of a row, or on top of a column header, to reorder; Esc cancels the drag
* Every change is one ordinary edit, so Ctrl+Z undoes it. Tables whose pipes are already aligned stay aligned; compact tables stay compact
* Only tables written in the page are editable; query and Lua output is left alone

```space-style
.sb-table-widget td,
.sb-table-widget th {
  cursor: text;
}
.sb-table-widget td a,
.sb-table-widget th a {
  cursor: pointer;
}
.sb-table-widget .sb-cell-editing {
  padding: 0 !important;
}
.sb-cell-editor {
  display: block;
  width: 100%;
  box-sizing: border-box;
  margin: 0;
  padding: 2px 4px;
  font: inherit;
  line-height: inherit;
  color: inherit;
  background: var(--root-background-color);
  border: 1px solid var(--ui-accent-color);
  border-radius: var(--ds-radius);
  outline: none;
  resize: none;
  overflow: hidden;
}
.sb-table-handle {
  position: fixed;
  z-index: 50;
  display: none;
  width: 18px;
  height: 18px;
  padding: 0;
  align-items: center;
  justify-content: center;
  color: var(--ds-muted);
  background: var(--ds-surface);
  border: 1px solid var(--ds-edge);
  border-radius: var(--ds-radius);
  cursor: pointer;
}
.sb-table-handle.is-visible {
  display: flex;
}
.sb-table-handle:hover {
  color: var(--root-color);
}
.sb-table-handle svg {
  width: 14px;
  height: 14px;
}
.sb-table-handle.is-grip {
  cursor: grab;
  touch-action: none;
}
.sb-table-handle.is-col svg {
  transform: rotate(90deg);
}
body.sb-table-dragging,
body.sb-table-dragging * {
  cursor: grabbing !important;
  user-select: none;
}
/* decoration, not text contrast: the row or column being carried */
.sb-table-widget .sb-table-dragged {
  opacity: 0.4;
}
.sb-table-drop {
  position: fixed;
  z-index: 55;
  pointer-events: none;
  background: var(--ui-accent-color);
}
.sb-table-menu {
  position: fixed;
  z-index: 60;
  min-width: 190px;
  padding: 4px 0;
  font-size: var(--ds-font-sm);
  color: var(--root-color);
  background: var(--ds-surface);
  border: 1px solid var(--ds-edge);
  border-radius: var(--ds-radius);
  box-shadow: var(--ds-elevation-2);
}
.sb-table-menu .sb-table-menu-head {
  padding: 6px 12px 2px;
  font-size: var(--ds-font-xs);
  color: var(--ds-muted);
}
.sb-table-menu button {
  display: block;
  width: 100%;
  padding: 4px 12px;
  text-align: left;
  font: inherit;
  color: inherit;
  background: none;
  border: 0;
  cursor: pointer;
}
.sb-table-menu button:hover,
.sb-table-menu button:focus-visible {
  background: color-mix(in srgb, var(--ui-accent-color) 14%, transparent);
  outline: none;
}
.sb-table-menu button.is-current::after {
  content: " ✓";
}
.sb-table-menu button.is-danger {
  color: var(--ds-danger-fg, var(--root-color));
}
.sb-table-menu hr {
  margin: 4px 0;
  border: 0;
  border-top: 1px solid var(--ds-edge);
}
```

```space-lua
local doc = js.window.document
if not doc.getElementById("sb-table-edit") then
  local s = doc.createElement("script")
  s.id = "sb-table-edit"
  s.innerHTML = [[
(function () {
  const view = () => window.client && window.client.editorView;
  const notify = (msg) => { try { window.client.flashNotification(msg, "error"); } catch (e) { console.warn(msg); } };
  const len = s => Array.from(s).length;
  const ROW_RE = /^\s*\|/;
  const DELIM_RE = /^\s*\|?(\s*:?-+:?\s*\|)*\s*:?-+:?\s*\|?\s*$/;
  const DELIM = { "": "---", left: ":---", center: ":---:", right: "---:" };

  // ---- markdown <-> model --------------------------------------------------
  function splitRow(line) {
    let s = line.trim();
    if (s.startsWith("|")) s = s.slice(1);
    if (s.endsWith("|") && !s.endsWith("\\|")) s = s.slice(0, -1);
    const cells = [];
    let cur = "";
    for (let i = 0; i < s.length; i++) {
      if (s[i] === "\\" && s[i + 1] === "|") { cur += "\\|"; i++; continue; }
      if (s[i] === "|") { cells.push(cur.trim()); cur = ""; continue; }
      cur += s[i];
    }
    cells.push(cur.trim());
    return cells;
  }
  const alignOf = c => c.startsWith(":") && c.endsWith(":") ? "center"
    : c.endsWith(":") ? "right" : c.startsWith(":") ? "left" : "";

  // The table whose first line holds `pos`, read from the document itself
  function readTable(pos) {
    const v = view();
    if (!v || v.state.readOnly) return null;
    const doc = v.state.doc;
    if (pos < 0 || pos > doc.length) return null;
    const lines = [];
    for (let i = doc.lineAt(pos).number; i <= doc.lines; i++) {
      const l = doc.line(i);
      if (!ROW_RE.test(l.text)) break;
      lines.push(l);
    }
    if (lines.length < 2 || !DELIM_RE.test(lines[1].text)) return null;
    const align = splitRow(lines[1].text).map(alignOf);
    const rows = [lines[0], ...lines.slice(2)].map(l => {
      const c = splitRow(l.text);
      while (c.length < align.length) c.push("");
      // remembered so an untouched row is written back byte for byte
      c.src = l.text;
      c.was = c.join("|");
      return c;
    });
    const widths = lines.map(l => len(l.text));
    return {
      from: lines[0].from, to: lines[lines.length - 1].to,
      indent: lines[0].text.match(/^\s*/)[0], align, rows,
      delimSrc: lines[1].text, alignWas: align.join("|"),
      // aligned pipes in the source are kept aligned; compact tables stay compact
      padded: lines.length > 2 && widths.every(w => w === widths[0]) &&
        lines.some(l => /\S {2,}\|/.test(l.text)),
    };
  }

  function writeTable(t) {
    const cols = t.align.length;
    const w = [];
    t.rows.forEach(r => r.forEach((s, c) => { w[c] = Math.max(w[c] || 3, len(s)); }));
    const pad = (s, c) => {
      const gap = w[c] - len(s), a = t.align[c];
      const left = a === "right" ? gap : a === "center" ? Math.floor(gap / 2) : 0;
      return " ".repeat(left) + s + " ".repeat(gap - left);
    };
    const cell = (s, c) => t.padded ? " " + pad(s, c) + " " : s === "" ? " " : " " + s + " ";
    const delim = (a, c) => {
      if (!t.padded) return DELIM[a];
      const d = "-".repeat(w[c] - (a === "center" ? 2 : a ? 1 : 0));
      return " " + (a === "center" || a === "left" ? ":" : "") + d +
        (a === "center" || a === "right" ? ":" : "") + " ";
    };
    // a compact table only rewrites the rows that changed; a padded one re-pads every row
    const row = r => !t.padded && r.src != null && r.join("|") === r.was ? r.src
      : t.indent + "|" + r.map(cell).join("|") + "|";
    const delimLine = !t.padded && t.align.join("|") === t.alignWas ? t.delimSrc
      : t.indent + "|" + t.align.slice(0, cols).map(delim).join("|") + "|";
    return [row(t.rows[0]), delimLine, ...t.rows.slice(1).map(row)].join("\n");
  }

  const escapeCell = v => v.replace(/\r?\n/g, " ").trim()
    .replace(/(\\*)\|/g, (m, bs) => bs.length % 2 ? m : bs + "\\|");
  const unescapeCell = s => s.replace(/\\\|/g, "|");

  // Re-read the table at `pos`, let fn change the model, write it back as one edit.
  // fn returns false to abort; `rows` guards against the source having moved on.
  function mutate(loc, fn) {
    const v = view(), t = readTable(loc.pos);
    if (!t || t.rows.length !== loc.rows) { notify("Table changed underneath the edit; nothing saved"); return false; }
    if (fn(t) === false) return false;
    const text = writeTable(t);
    if (text !== v.state.sliceDoc(t.from, t.to)) {
      const tr = v.state.update({ changes: { from: t.from, to: t.to, insert: text }, userEvent: "input" });
      v.dispatch(tr);
      lastChanges = tr.changes;
    }
    return true;
  }

  let lastChanges = null;

  // ---- DOM <-> source position -----------------------------------------------
  function tablePos(table) {
    const v = view(), w = table && table.closest(".sb-table-widget");
    if (!v || !w || !v.contentDOM.contains(w)) return null;
    try { return v.posAtDOM(w, 0); } catch (e) { return null; }
  }
  function locate(cell) {
    const table = cell.closest("table"), tr = cell.parentElement;
    const pos = tablePos(table);
    if (pos == null) return null;
    const t = readTable(pos);
    if (!t || t.rows.length !== table.rows.length) return null;
    return { pos, rows: table.rows.length, cols: t.align.length,
      r: Array.prototype.indexOf.call(table.rows, tr),
      c: Array.prototype.indexOf.call(tr.cells, cell) };
  }
  function findTable(pos) {
    const v = view();
    if (!v) return null;
    for (const w of v.contentDOM.querySelectorAll(".sb-table-widget")) {
      let p;
      try { p = v.posAtDOM(w, 0); } catch (e) { continue; }
      const t = w.querySelector("table");
      if (p === pos && t) return t;
    }
    return null;
  }
  // CodeMirror swaps the widget synchronously but fills the table asynchronously
  function whenRendered(pos, rows, cb) {
    let tries = 0;
    const tick = () => {
      const t = findTable(pos);
      if (t && (rows == null || t.rows.length === rows)) return cb(t);
      if (++tries < 120) requestAnimationFrame(tick);
    };
    tick();
  }

  // ---- cell editor -----------------------------------------------------------
  let active = null;

  function openAt(pos, r, c) {
    whenRendered(pos, null, table => {
      const row = table.rows[r], cell = row && row.cells[c];
      if (cell) open(cell);
    });
  }

  function open(cell) {
    const loc = locate(cell);
    if (!loc) return;
    const src = (readTable(loc.pos).rows[loc.r] || [])[loc.c] || "";
    const ta = document.createElement("textarea");
    ta.className = "sb-cell-editor";
    ta.rows = 1;
    ta.spellcheck = true;
    ta.value = unescapeCell(src);
    const kept = document.createDocumentFragment();
    while (cell.firstChild) kept.appendChild(cell.firstChild);
    cell.classList.add("sb-cell-editing");
    cell.appendChild(ta);
    const fit = () => { ta.style.height = "auto"; ta.style.height = ta.scrollHeight + "px"; };
    active = { loc, src, ta, cell, kept };
    hideHandle();

    for (const ev of ["mousedown", "mouseup", "click", "dblclick", "keyup", "keypress",
      "beforeinput", "input", "paste", "copy", "cut", "compositionstart", "compositionend"])
      ta.addEventListener(ev, e => e.stopPropagation());
    ta.addEventListener("input", fit);
    ta.addEventListener("blur", () => { if (active && active.ta === ta) close(true, true); });
    ta.addEventListener("keydown", e => {
      e.stopPropagation();
      if (e.isComposing) return;
      if (e.key === "Escape") { e.preventDefault(); close(false); }
      else if (e.key === "Enter" && !e.shiftKey) { e.preventDefault(); close(true); }
      else if (e.key === "Tab") { e.preventDefault(); step(e.shiftKey ? -1 : 1); }
    });
    ta.focus();
    ta.setSelectionRange(ta.value.length, ta.value.length);
    fit();
  }

  // Put the rendered cell back, then save if the text changed. Returns the row count after.
  function close(save, keepFocus) {
    const a = active;
    if (!a) return null;
    active = null;
    const value = escapeCell(a.ta.value);
    a.ta.remove();
    a.cell.classList.remove("sb-cell-editing");
    a.cell.appendChild(a.kept);
    if (save && value !== a.src) {
      mutate(a.loc, t => {
        if ((t.rows[a.loc.r][a.loc.c] || "") !== a.src) { notify("Cell changed underneath the edit; nothing saved"); return false; }
        while (t.rows[a.loc.r].length <= a.loc.c) t.rows[a.loc.r].push("");
        t.rows[a.loc.r][a.loc.c] = value;
      });
    }
    if (!keepFocus) { const v = view(); if (v) v.focus(); }
    return a.loc;
  }

  function step(dir) {
    const loc = close(true, true);
    if (!loc) return;
    let r = loc.r, c = loc.c + dir;
    if (c >= loc.cols) { c = 0; r++; }
    if (c < 0) { c = loc.cols - 1; r--; }
    if (r < 0) { const v = view(); if (v) v.focus(); return; }
    if (r >= loc.rows) {
      if (!mutate(loc, t => { t.rows.push(t.align.map(() => "")); })) return;
      whenRendered(loc.pos, loc.rows + 1, () => openAt(loc.pos, r, 0));
      return;
    }
    openAt(loc.pos, r, c);
  }

  // Plain click on a cell edits it; Alt/Ctrl/Meta/Shift-click and links keep SilverBullet's behaviour
  let swallowClick = false;
  window.addEventListener("mousedown", e => {
    if (e.button !== 0 || e.altKey || e.ctrlKey || e.metaKey || e.shiftKey) return;
    const cell = e.target.closest && e.target.closest(".sb-table-widget td, .sb-table-widget th");
    if (!cell || cell.classList.contains("sb-cell-editing")) return;
    if (document.body.classList.contains("sb-col-resize-hover")) return;
    if (e.target.closest("a, button, input, textarea, select, label, .sb-task-state")) return;
    const loc = locate(cell);
    if (!loc) return;
    e.preventDefault();
    e.stopPropagation();
    swallowClick = true;
    setTimeout(() => { swallowClick = false; }, 600);
    closeMenu();
    if (active) {
      // saving the open cell may shift this table, so map its position through that edit
      lastChanges = null;
      close(true, true);
      openAt(lastChanges ? lastChanges.mapPos(loc.pos) : loc.pos, loc.r, loc.c);
    } else open(cell);
  }, true);
  window.addEventListener("click", e => {
    if (!swallowClick) return;
    swallowClick = false;
    if (e.target.closest && e.target.closest(".sb-table-widget")) { e.preventDefault(); e.stopPropagation(); }
  }, true);

  // ---- row / column menu -----------------------------------------------------
  const ICON = '<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><circle cx="12" cy="12" r="1"/><circle cx="12" cy="5" r="1"/><circle cx="12" cy="19" r="1"/></svg>';
  const GRIP = '<svg viewBox="0 0 24 24" fill="currentColor"><circle cx="9" cy="6" r="1.5"/><circle cx="15" cy="6" r="1.5"/><circle cx="9" cy="12" r="1.5"/><circle cx="15" cy="12" r="1.5"/><circle cx="9" cy="18" r="1.5"/><circle cx="15" cy="18" r="1.5"/></svg>';
  const mkHandle = (cls, title, svg) => {
    const b = document.createElement("button");
    b.className = "sb-table-handle" + cls;
    b.title = title;
    b.tabIndex = -1;
    b.innerHTML = svg;
    document.body.appendChild(b);
    return b;
  };
  const handle = mkHandle("", "Table row / column actions", ICON);
  const rowGrip = mkHandle(" is-grip", "Drag to move row", GRIP);
  const colGrip = mkHandle(" is-grip is-col", "Drag to move column", GRIP);
  const handles = [handle, rowGrip, colGrip];
  let hoverCell = null, menu = null, drag = null, hideTimer = 0;

  const place = (el, x, y) => {
    el.style.left = Math.max(0, x) + "px";
    el.style.top = Math.max(0, y) + "px";
    el.classList.add("is-visible");
  };
  function hideHandle() { handles.forEach(h => h.classList.remove("is-visible")); hoverCell = null; }
  function showHandle(cell) {
    const table = cell.closest("table"), tr = cell.parentElement;
    const r = cell.getBoundingClientRect(), tb = table.getBoundingClientRect();
    place(handle, r.right - 21, r.top + Math.max(2, (r.height - 18) / 2));
    // the header row is the header: it has no row grip
    if (tr.rowIndex > 0) {
      const rr = tr.getBoundingClientRect();
      place(rowGrip, tb.left - 21, rr.top + (rr.height - 18) / 2);
    } else rowGrip.classList.remove("is-visible");
    const head = table.rows[0].cells[cell.cellIndex];
    if (head) {
      const h = head.getBoundingClientRect();
      place(colGrip, h.left + (h.width - 18) / 2, h.top - 10);
    } else colGrip.classList.remove("is-visible");
    hoverCell = cell;
  }
  const cancelHide = () => { clearTimeout(hideTimer); hideTimer = 0; };

  // the grips sit outside the cell, so leaving the table hides them only after a pause
  document.addEventListener("mousemove", e => {
    if (menu || active || drag) return;
    if (handles.some(h => h.contains(e.target))) return cancelHide();
    const cell = e.target.closest && e.target.closest(".sb-table-widget td, .sb-table-widget th");
    if (!cell || document.body.classList.contains("sb-col-resizing")) {
      if (hoverCell && !hideTimer) hideTimer = setTimeout(() => { hideTimer = 0; hideHandle(); }, 250);
      return;
    }
    cancelHide();
    if (cell !== hoverCell && tablePos(cell.closest("table")) != null) showHandle(cell);
  }, { passive: true });
  window.addEventListener("scroll", () => { if (!drag) { hideHandle(); closeMenu(); } }, true);

  // ---- drag to reorder ---------------------------------------------------------
  function startDrag(e, axis) {
    if (e.button !== 0 || !hoverCell) return;
    e.preventDefault();
    e.stopPropagation();
    const cell = hoverCell, table = cell.closest("table"), loc = locate(cell);
    if (!loc) return;
    const from = axis === "row" ? loc.r : loc.c;
    const carried = axis === "row" ? Array.of(table.rows[from])
      : Array.from(table.rows, tr => tr.cells[from]).filter(Boolean);
    carried.forEach(el => el.classList.add("sb-table-dragged"));
    const line = document.createElement("div");
    line.className = "sb-table-drop";
    document.body.appendChild(line);
    document.body.classList.add("sb-table-dragging");
    drag = { to: from };

    // drag.to is a slot: "before row/column i", or the count for "at the end"
    const move = ev => {
      const tb = table.getBoundingClientRect();
      if (axis === "row") {
        let to = table.rows.length;
        for (let i = 1; i < table.rows.length; i++) {
          const b = table.rows[i].getBoundingClientRect();
          if (ev.clientY < b.top + b.height / 2) { to = i; break; }
        }
        drag.to = to;
        const y = to < table.rows.length ? table.rows[to].getBoundingClientRect().top : tb.bottom;
        Object.assign(line.style, { left: tb.left + "px", top: (y - 1) + "px", width: tb.width + "px", height: "2px" });
      } else {
        const hc = Array.from(table.rows[0].cells);
        let to = hc.length;
        for (let i = 0; i < hc.length; i++) {
          const b = hc[i].getBoundingClientRect();
          if (ev.clientX < b.left + b.width / 2) { to = i; break; }
        }
        drag.to = to;
        const x = to < hc.length ? hc[to].getBoundingClientRect().left : tb.right;
        Object.assign(line.style, { left: (x - 1) + "px", top: tb.top + "px", width: "2px", height: tb.height + "px" });
      }
    };
    const finish = commit => {
      window.removeEventListener("pointermove", move);
      window.removeEventListener("pointerup", up);
      window.removeEventListener("keydown", esc, true);
      carried.forEach(el => el.classList.remove("sb-table-dragged"));
      line.remove();
      document.body.classList.remove("sb-table-dragging");
      const to = drag.to;
      drag = null;
      hideHandle();
      const dest = to > from ? to - 1 : to;
      if (!commit || dest === from) return;
      mutate(loc, t => {
        const moveIn = a => { while (a.length <= from) a.push(""); const x = a.splice(from, 1)[0]; a.splice(dest, 0, x); };
        if (axis === "row") moveIn(t.rows);
        else { t.rows.forEach(moveIn); moveIn(t.align); }
      });
    };
    const up = () => finish(true);
    const esc = ev => { if (ev.key === "Escape") { ev.preventDefault(); ev.stopPropagation(); finish(false); } };
    move(e);
    window.addEventListener("pointermove", move);
    window.addEventListener("pointerup", up);
    window.addEventListener("keydown", esc, true);
  }
  rowGrip.addEventListener("pointerdown", e => startDrag(e, "row"));
  colGrip.addEventListener("pointerdown", e => startDrag(e, "col"));
  for (const g of [rowGrip, colGrip]) g.addEventListener("click", e => { e.preventDefault(); e.stopPropagation(); });

  handle.addEventListener("mousedown", e => { e.preventDefault(); e.stopPropagation(); });
  handle.addEventListener("click", e => {
    e.preventDefault();
    e.stopPropagation();
    const cell = hoverCell, loc = cell && locate(cell);
    if (!loc) return hideHandle();
    const r = handle.getBoundingClientRect();
    openMenu(loc, r.right, r.bottom + 2);
  });

  function closeMenu() {
    if (!menu) return;
    menu.remove();
    menu = null;
    hideHandle();
  }
  document.addEventListener("mousedown", e => { if (menu && !menu.contains(e.target)) closeMenu(); }, true);
  document.addEventListener("keydown", e => { if (menu && e.key === "Escape") { e.preventDefault(); e.stopPropagation(); closeMenu(); } }, true);

  // (no destructuring swap: a double close-bracket would end the Lua long string)
  const swap = (a, i, j) => { while (a.length <= Math.max(i, j)) a.push(""); const x = a[i]; a[i] = a[j]; a[j] = x; };

  function openMenu(loc, x, y) {
    closeMenu();
    const { r, c, rows, cols } = loc;
    const head = r === 0;
    // then: [r, c] of the cell to edit afterwards, or nothing
    const act = (fn, nRows, then) => () => {
      closeMenu();
      if (!mutate(loc, fn)) return;
      if (then) whenRendered(loc.pos, nRows, () => openAt(loc.pos, then[0], then[1]));
    };
    const items = [
      ["Row"],
      !head && ["Insert row above", act(t => { t.rows.splice(r, 0, t.align.map(() => "")); }, rows + 1, [r, c])],
      ["Insert row below", act(t => { t.rows.splice(r + 1, 0, t.align.map(() => "")); }, rows + 1, [r + 1, c])],
      r > 1 && ["Move row up", act(t => { swap(t.rows, r, r - 1); })],
      !head && r < rows - 1 && ["Move row down", act(t => { swap(t.rows, r, r + 1); })],
      !head && ["Delete row", act(t => { t.rows.splice(r, 1); }), "is-danger"],
      ["Column"],
      ["Insert column left", act(t => { t.rows.forEach(row => { while (row.length < c) row.push(""); row.splice(c, 0, ""); }); t.align.splice(c, 0, ""); }, rows, [0, c])],
      ["Insert column right", act(t => { t.rows.forEach(row => { while (row.length <= c) row.push(""); row.splice(c + 1, 0, ""); }); t.align.splice(c + 1, 0, ""); }, rows, [0, c + 1])],
      c > 0 && ["Move column left", act(t => { t.rows.forEach(row => swap(row, c, c - 1)); swap(t.align, c, c - 1); })],
      c < cols - 1 && ["Move column right", act(t => { t.rows.forEach(row => swap(row, c, c + 1)); swap(t.align, c, c + 1); })],
      cols > 1 && ["Delete column", act(t => { t.rows.forEach(row => row.splice(c, 1)); t.align.splice(c, 1); }), "is-danger"],
      ["Align column"],
    ];
    const cur = (readTable(loc.pos) || { align: [] }).align[c] || "";
    const aligns = { Default: "", Left: "left", Center: "center", Right: "right" };
    for (const label in aligns) {
      const a = aligns[label];
      items.push([label, act(t => { t.align[c] = a; }), a === cur ? "is-current" : ""]);
    }

    menu = document.createElement("div");
    menu.className = "sb-table-menu";
    menu.setAttribute("role", "menu");
    let first = true;
    for (const it of items.filter(Boolean)) {
      if (it.length === 1) {
        if (!first) menu.appendChild(document.createElement("hr"));
        const h = document.createElement("div");
        h.className = "sb-table-menu-head";
        h.textContent = it[0];
        menu.appendChild(h);
      } else {
        const b = document.createElement("button");
        b.setAttribute("role", "menuitem");
        b.textContent = it[0];
        if (it[2]) b.className = it[2];
        b.addEventListener("mousedown", e => e.preventDefault());
        b.addEventListener("click", e => { e.preventDefault(); e.stopPropagation(); it[1](); });
        menu.appendChild(b);
      }
      first = false;
    }
    document.body.appendChild(menu);
    const m = menu.getBoundingClientRect();
    menu.style.left = Math.max(4, Math.min(x - m.width, innerWidth - m.width - 4)) + "px";
    menu.style.top = (y + m.height > innerHeight - 4 ? Math.max(4, y - m.height - 24) : y) + "px";
    handle.classList.add("is-visible");
  }
})();
]]
  doc.head.appendChild(s)
end
```
