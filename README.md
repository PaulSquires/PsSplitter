# CSplitter

An owner-drawn **splitter bar** control for FreeBASIC / Win32, built on the AfxNova
framework. Same family as `CVScrollBar` / `CHScrollBar` / `CStatusBar` / `CTabBar` /
`CTextBox` / `CMenuBar`. Built to replace tiko's three hand-rolled splitters (the help
viewer's left/right bar, the editor's split-document bar, the Output panel's bar), each of
which was a virtual rect hit-tested inside a form's `WM_MOUSEMOVE` with the drag state
parked in globals.

**The control does not own, create, size, or know about the pane windows.** It maintains
exactly one number — the splitter's position — and tells the host when the user changes it.
The host keeps full control of its own layout.

## Design (shared with the control family)

- One real `HWND` per instance — callers `SetWindowPos` it directly; no opaque handle type.
- All per-instance state lives in a `CSPLITTER` TYPE stored in the CWindow UserData area,
  reached through a private `GetSplitterPointer(hwnd)` accessor. Any number of instances
  coexist.
- **The position is the bar's leading edge in the PARENT's client coordinates** — `x` for a
  vertical bar, `y` for a horizontal one. Min/max limits live in the same space.
- **Live drag; the control moves itself.** Each mouse move clamps the new position,
  `SetWindowPos`es the control, then notifies. The host lays out the two panes and nothing
  else. Re-placing the bar from the host's own layout pass is a harmless no-op.
- **Limits are the host's to compute.** The control never derives them from the parent's
  client rect, because it cannot know what else lives there (tiko's help viewer parks a
  scrollbar between the pane and the bar). Recompute and push them from your `WM_SIZE`.
- **Programmatic setters are silent; only user interaction notifies.** `SetPos`/`SetRange`
  never fire the position callback — that's the host telling the control where it put the
  splitter, so a host can call them from inside its own callback without re-entering.
- **The position never lies.** `WM_WINDOWPOSCHANGED` re-syncs the stored position from the
  window's actual rect whenever the *host* moves the bar, so `GetPos` and reality cannot
  drift apart. That re-sync notifies nothing — it is an observation, not a change.
- Takes **mouse capture** for the drag (it earns it: the gesture consumes the down→up
  pairing) and pays the full price — `SetCapture` before any host callback, the message
  callback's result ignored for `WM_LBUTTONUP`, `WM_CAPTURECHANGED` closes the gesture,
  `WM_NCDESTROY` releases.
- **Capture lost mid-drag keeps the position** and fires `END`: the host has already
  applied every intermediate position and has nothing to undo.
- Sets `IDC_SIZEWE` / `IDC_SIZENS` from `WM_SETCURSOR` per orientation, and holds it for
  the whole drag (Windows routes `WM_SETCURSOR` to the capture window).
- Hover tracking uses `TrackMouseEvent` plus a 100 ms poll timer safety net
  (`WM_MOUSELEAVE` is not reliably delivered on fast exits). Hover is **pinned** for the
  duration of a drag, since the cursor legitimately roams outside the client under capture.
- One `CBufferPaint` per `WM_PAINT`; built-in flat paint, or a host paint callback that
  overrides rendering entirely. No host globals — colors are passed in.
- **No keyboard support** by design. The host owns focus; nudging the splitter from a key
  is a `CSplitter_SetPos` call away.

## API

```freebasic
declare function CSplitter_Create( byval hWndParent as HWND, byval CtrlID as integer, _
                                   byval nOrientation as integer = CSPLITTER_VERTICAL ) as HWND
declare function CSplitter_GetOrientation( byval hSplit as HWND ) as integer

declare function CSplitter_GetPos( byval hSplit as HWND ) as integer
declare sub      CSplitter_SetPos( byval hSplit as HWND, byval nPosition as integer )   ' silent
declare sub      CSplitter_SetRange( byval hSplit as HWND, byval nMin as integer, byval nMax as integer )
declare function CSplitter_GetMin( byval hSplit as HWND ) as integer
declare function CSplitter_GetMax( byval hSplit as HWND ) as integer

declare function CSplitter_IsHot( byval hSplit as HWND ) as boolean
declare function CSplitter_IsDragging( byval hSplit as HWND ) as boolean

declare function CSplitter_GetBackColor( byval hSplit as HWND ) as COLORREF
declare sub      CSplitter_SetColors( byval hSplit as HWND, byval backclr as COLORREF, byval hotclr as COLORREF )

declare sub      CSplitter_SetPaintCallback( byval hSplit as HWND, byval usersub as SPL_PaintCallbackSub )
declare sub      CSplitter_SetMessageCallback( byval hSplit as HWND, byval userfunc as SPL_MessageCallbackFunc )
declare sub      CSplitter_SetPosChangedCallback( byval hSplit as HWND, byval usersub as SPL_PosChangedCallbackSub )
```

Orientation is **fixed at creation** and named for the *bar's own axis*:

```freebasic
enum CSPLITTER_ORIENTATION
    CSPLITTER_VERTICAL   = 0   ' vertical bar,   left/right panes -> IDC_SIZEWE, position = x
    CSPLITTER_HORIZONTAL = 1   ' horizontal bar, top/bottom panes -> IDC_SIZENS, position = y
end enum
```

A host that switches split modes destroys the control and creates the other one — cheaper
than reasoning about a position whose meaning changed axis underneath a stale range.

Callbacks:

```freebasic
' Paint override: receives the control's double buffer plus its rect and state.
type SPL_PaintCallbackSub as sub( byval p as CSPLITTER_PAINTINFO ptr )

' Observe mouse messages (incl. WM_LBUTTONDBLCLK and WM_SETCURSOR). TRUE = handled,
' suppress default handling. The result is IGNORED for WM_LBUTTONUP -- that message is the
' mouse capture's exit and suppressing it would strand the capture.
type SPL_MessageCallbackFunc as function( byval m as CSPLITTER_MESSAGEINFO ptr ) as boolean

' The user moved the splitter. nPhase is CSPLITTER_PHASE_BEGIN / _MOVE / _END.
type SPL_PosChangedCallbackSub as sub( byval hSplitter as HWND, byval newPos as integer, _
                                       byval nPhase as integer )
```

`BEGIN` fires on the grab (with the current position), `MOVE` only when the position
actually changes, `END` on button-up **or** capture loss. Use `BEGIN`/`END` to bracket work
that is too expensive per-pixel — tiko hides the editor's horizontal scrollbars for the
duration of a splitter drag.

The control is created hidden and zero-sized. The size you give it along the drag axis
**is** the grab area — there is no separate hit margin; paint a thinner line inside it if
you want a thin look.

Callback typedef namespace note: tiko includes the whole control family, so typedef names
are global. This control claims the `SPL_` prefix.

## Usage sketch

```freebasic
hSplit = CSplitter_Create( hParent, IDC_SPLIT, CSPLITTER_VERTICAL )
CSplitter_SetColors( hSplit, barclr, barhotclr )
CSplitter_SetPosChangedCallback( hSplit, @OnSplitterMoved )

' on layout (your WM_SIZE): compute the limits, place the bar, place the panes
sub Layout()
    dim as RECT rc : GetClientRect( hParent, @rc )
    CSplitter_SetRange( hSplit, minLeftW, rc.right - minRightW - barW )
    SetWindowPos( hSplit, 0, CSplitter_GetPos(hSplit), 0, barW, rc.bottom, _
                  SWP_NOZORDER or SWP_SHOWWINDOW )
    PositionPanes()
end sub

' the user dragged it: the bar has already moved itself
sub OnSplitterMoved( byval hSplitter as HWND, byval newPos as integer, byval nPhase as integer )
    select case nPhase
        case CSPLITTER_PHASE_BEGIN : HideExpensiveChrome()
        case CSPLITTER_PHASE_MOVE  : PositionPanes()
        case CSPLITTER_PHASE_END   : PositionPanes() : ShowExpensiveChrome() : SaveToConfig( newPos )
    end select
end sub

' restoring a saved layout (silent -- repaint your panes yourself):
CSplitter_SetPos( hSplit, savedPos )
```

Double-click to collapse/restore a pane is the host's job; claim the message:

```freebasic
function OnSplitterMessage( byval m as CSPLITTER_MESSAGEINFO ptr ) as boolean
    if m->uMsg <> WM_LBUTTONDBLCLK then return false
    CSplitter_SetPos( m->hSplitter, iif( collapsed, savedPos, CSplitter_GetMax(m->hSplitter) ) )
    return true          ' claimed: no drag starts
end function
```

## Building the demo

```
fbc64.exe -i "C:\dev" main.bas
```

(Needs the AfxNova framework at `C:\dev\AfxNova`.) `main.exe` is a console-subsystem demo:
three painted panes separated by a vertical and a horizontal splitter, with position
notifications traced to the console. The vertical bar uses the built-in painter, the
horizontal one a custom paint callback plus double-click collapse.

Run with environment variable `CSPLITTER_SELFTEST=1` for a headless self-test, then exit.
It asserts position↔window-rect agreement, clamping (including an inverted range), the
silence of the programmatic setters, the `WM_WINDOWPOSCHANGED` re-sync, the drag mapping at
both clamps and mid-track (including that clamping is **not sticky**), the
`BEGIN / MOVE… / END` notification sequence with no duplicate `MOVE`, and capture-loss
behaviour. The drag assertions drive the real `WndProc` — the cursor is warped with
`SetCursorPos` (and restored afterwards) so the arithmetic under test is exactly what a
real drag runs.
