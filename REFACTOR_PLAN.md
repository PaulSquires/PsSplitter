# CSplitter — design rationale and tiko migration mapping

Status: **control complete** (2026-07-21). Standalone repo builds clean, self-test green.
tiko migration **not started** — the mapping below is the plan, not a record.

---

## Why this exists

tiko hand-rolls a splitter **five** times. None of them is a window; each is a virtual rect
hit-tested inside some form's `WM_MOUSEMOVE`, with the drag state living in one shared
global, `gApp.bDragActive` ([clsApp.bi:29](../tiko/src/clsApp.bi)):

| Site | Bar | How the rect is found | Extra state |
|---|---|---|---|
| [frmHelpViewer.inc:98](../tiko/src/frmHelpViewer.inc) | vertical, left/right panes | `frmHelpViewer_CalcSplitRect` | `gHelpViewer.xDeltaSplitter`, `ptSplitPrev` |
| [frmMain.inc:433](../tiko/src/frmMain.inc) | editor split, both axes | `frmMain_CalcSplitRect(pDoc)` → `pDoc->rcSplitButton` | `pDoc->EditorSplitMode`, `SplitX`/`SplitY`, `doubleClickReceived` |
| [frmOutput.inc:400](../tiko/src/frmOutput.inc) | horizontal, output panel top edge | inline: window rect, top `SPLITSIZE` px | `gApp.idTimerOutputPanel`, `gApp.doubleClickReceived` |
| [frmPanel.inc:255](../tiko/src/frmPanel.inc) | vertical, explorer panel edge | inline: window rect, 3 px edge | `gApp.hwndPanel`, `gConfig.ExplorerPositionRight` |
| [frmListView.inc:138](../tiko/src/frmListView.inc) | column-ish drag | inline | — |

Costs of that shape, all of which the control removes:

- **One global for five drags.** `gApp.bDragActive` is only safe because at most one drag
  can be live; nothing enforces it, and a missed reset in one form leaks into another
  (`frmPanel` guards with a *second* global, `gApp.hwndPanel`, precisely because of this).
- **Cursor setting is scattered** through each host's `WM_MOUSEMOVE`, each re-deriving the
  hit rect and calling `WindowFromPoint` to make sure it is really over the right window.
- **Delta accumulation.** `frmHelpViewer` tracks `ptSplitPrev` and applies `xDelta`; that is
  the incremental scheme the control deliberately does not use, and it is why the help
  viewer's bar feels sticky against its clamps.
- **No hover state at all** — none of the five highlights on hover, because there is nowhere
  to keep the flag.
- Each site re-implements min/max clamping in its own layout function, in different units.

---

## Design decisions (do not re-litigate)

- **The position is the bar's leading edge in the PARENT's client coordinates** (`x` for a
  vertical bar, `y` for a horizontal one), *not* the first pane's size and *not* a fraction.
  Pane-size was rejected because it assumes the panes tile the parent's whole client area,
  which is false for the help viewer (a `CVScrollBar` sits between the left pane and the
  bar). A fraction was rejected because the whole family is integer-pixel.
- **Live drag, and the control moves itself.** Notify-only was the alternative (host's
  layout pass is the single source of truth); rejected because the bar visibly lags a host
  that defers layout, and re-placing the bar from the host's layout is a harmless no-op
  anyway. A deferred "ghost bar" drag was rejected as a behaviour regression — all five
  tiko sites resize live today.
- **One callback with a BEGIN / MOVE / END phase**, not a bare position-changed callback.
  Evidence it is needed: `frmMain_HideHorizScrollBars` exists solely to bracket an editor
  splitter drag.
- **Limits are absolute and host-supplied** (`SetRange(nMin, nMax)`), not derived from the
  parent client rect and not expressed as min pane sizes. Same reason as the position: the
  control cannot know what else lives in the parent's client area.
- **Capture loss keeps the position** and fires `END`. Reverting would snap the panes back
  for a reason the host never asked for, and every intermediate position has already been
  applied.
- **Orientation is fixed at creation**, with a getter but no setter. Flipping it would
  change what the stored position *means* (x → y) while the range still described the old
  axis; destroy-and-recreate is cheaper to reason about. Note this narrows the original
  "Set/Get orientation" ask — deliberate.
- **Double-click is routed through the message callback**, not given its own callback. The
  control has no concept of a collapsed pane; the host claims `WM_LBUTTONDBLCLK`, calls
  `CSplitter_SetPos`, and no drag starts.
- **No keyboard support.** Consequence worth recording: the family's hover-clear poll timer
  is safe here *because* hover is the only writer of the highlight. If a keyboard-driven
  highlight is ever added, it needs the ownership tag from the CMenuBar learning
  (`Learnings.md`), or the poll will erase it every 100 ms.
- **The whole client rect is the grab area.** No separate hit margin: the host sizes the
  control to the grab width it wants and paints a thinner line inside it (the demo's
  horizontal bar does exactly that).

### Traps handled, with their sources

- `WM_LBUTTONDBLCLK` *substitutes* for the second `WM_LBUTTONDOWN` under `CS_DBLCLKS`, so it
  runs the same capture bookkeeping; the release is guarded by `GetCapture() = hwnd`.
- The message callback's result is **ignored for `WM_LBUTTONUP`** — that up is the capture's
  exit.
- `EndDrag` is a no-op when no drag is live, so the `WM_CAPTURECHANGED` that `ReleaseCapture`
  itself generates cannot fire a second `END`.
- Hover is **pinned** for the whole gesture and both clear paths (`WM_MOUSELEAVE`, the poll
  timer) stand down while dragging.
- The drag reads the cursor with `GetCursorPos` + `ScreenToClient(parent)`, never from
  `lParam` — `lParam` is relative to a window being moved mid-gesture, and delta
  accumulation makes the clamps sticky.
- A member named `hCursor` would shadow the `HCURSOR` *type* (FreeBASIC is case-insensitive)
  exactly as `hFont`/`HFONT` does in CStatusBar; hence `hBarCursor`.

---

## tiko migration mapping

The shape change to plan for: tiko's splitters are **edges of existing windows** (grab the
top 6 px of `frmOutput`, the right 3 px of `frmPanel`). CSplitter is a **separate window
between** the panes, so each migration reserves a bar-width gap in the host's layout and
shrinks the neighbouring pane by that much. That is a layout edit, not just a call swap.

**Common to every site**

1. `hSplit = CSplitter_Create( hHost, IDC_…, CSPLITTER_VERTICAL|_HORIZONTAL )`, colors from
   the theme, `SetPosChangedCallback`.
2. In the host's existing `PositionWindows`: compute `nMin`/`nMax`, `CSplitter_SetRange`,
   `SetWindowPos` the bar, then place the panes around it.
3. Delete the site's `WM_LBUTTONDOWN` / `WM_LBUTTONUP` / cursor-setting and rect-hit-testing
   code, and its use of `gApp.bDragActive` (and `gApp.hwndPanel`).
4. Restore a saved layout with `CSplitter_SetPos` — it is silent, so call the host's own
   layout/repaint afterwards.

**Per site**

- **frmHelpViewer** — the easy one, do it first. `frmHelpViewer_CalcSplitRect` is deleted
  outright; `xDeltaSplitter`/`ptSplitPrev` go with it, and
  `frmHelpViewer_PositionWindows`'s `xDelta` branch collapses to reading
  `CSplitter_GetPos`. Note the left `CVScrollBar` sits *between* the pane and the bar — the
  reason the position is a parent-client coordinate rather than a pane width.
- **frmMain (editor split)** — one control per split mode, created on demand:
  `SplitLeftRight` → `CSPLITTER_VERTICAL`, `SplitTopBottom` → `CSPLITTER_HORIZONTAL`
  (orientation is fixed at creation, so switching modes destroys and recreates). Delete
  `frmMain_CalcSplitRect` and the `pDoc->rcSplitButton` fill in `frmMain_OnPaint` — the bar
  paints itself now. `pDoc->SplitX`/`SplitY` become the persisted position, pushed back with
  `CSplitter_SetPos` on document activation. Hook `frmMain_HideHorizScrollBars` to
  `CSPLITTER_PHASE_BEGIN` and the re-show to `_END`.
- **frmOutput** — `CSPLITTER_HORIZONTAL` along the panel's top edge. Its single/double-click
  timer dance (`gApp.idTimerOutputPanel`, `gApp.doubleClickReceived`) is replaced by claiming
  `WM_LBUTTONDBLCLK` in a message callback: no timer, no discrimination needed, because the
  control's own `CS_DBLCLKS` class does it. `ShowOutputPanelMinimized` becomes the collapsed
  flag the callback toggles.
- **frmPanel** — `CSPLITTER_VERTICAL`, on whichever side `gConfig.ExplorerPositionRight`
  puts it; the min width (`ScaleX(236)`) becomes `nMin` (or `nMax`, mirrored, when the panel
  is docked right). `gApp.hwndPanel` disappears with `bDragActive`.
- **frmListView** — inspect before migrating; it may be a column drag rather than a pane
  splitter, in which case it belongs to `CColumnHeader`, not here.

Afterwards `gApp.bDragActive`, `gApp.hwndPanel`, `gApp.idTimerOutputPanel`,
`gApp.doubleClickReceived` and `clsDocument.doubleClickReceived` should all be deletable.
Grep for them as the completion check.

---

## Verification (what was actually done)

- Builds clean, zero warnings:
  `fbc64.exe -i "C:\dev" main.bas` (FreeBASIC 1.10.1, 64-bit).
- `CSPLITTER_SELFTEST=1 main.exe` — 40 assertions, 0 failures. Covers position↔window-rect
  agreement, clamping (including an inverted range), silence of the programmatic setters,
  the `WM_WINDOWPOSCHANGED` re-sync, the drag mapping at both clamps and mid-track including
  the non-sticky property, duplicate-`MOVE` suppression, the `BEGIN/MOVE…/END` sequence, and
  capture-loss behaviour. Drag assertions drive the real `WndProc` via `SetCursorPos` +
  `SendMessage`, so the arithmetic under test is the production path.
- **Not verified by the self-test**, by construction: real-mouse feel, the actual cursor
  *shape* (`WM_SETCURSOR` is never exercised by a synthetic drag), hover highlight
  appearance, the double-click collapse path, and mixed-DPI/multi-monitor behaviour. Those
  are the demo's manual checklist.
