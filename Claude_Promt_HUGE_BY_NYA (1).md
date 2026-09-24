Follow the promt and make the menu based on the picture provided by user!, make sure to make it in HTML.

# Dear ImGui → HTML/CSS/JS Clone: Implementation Specification v4

This document is the authoritative implementation specification for the project. It supersedes
any earlier v2/v3 review notes and is the version to hand directly to an implementation team or
coding agent. It is organized as the final implementation contract needed to prevent ambiguity
and accidental API drift.

---

## 1. Executive Summary — What Was Wrong With v2

v2 already did real work correcting v1's color/spacing numbers against ImGui's actual
`StyleColorsDark()`/`ImGuiStyle()` defaults, and it correctly identified the retained-vs-immediate
architecture question. But audited as an *implementation-ready* spec, it has one **contradiction
that would silently break the single most basic interaction (dragging a window)**, and it is
missing the internal machinery that makes "immediate mode" actually work in a browser rather
than just being a slogan.

**The critical bug:** v2 §11's own reference API example calls
`im.beginWindow('Settings', { x: 20, y: 20, w: 300 })` from inside the per-frame `drawUI()`
function that runs every single frame. §6.17 separately requires windows to be draggable by
their title bar. Under v2's own example, dragging would be immediately overwritten the very
next frame, because the caller re-asserts `x: 20, y: 20` every frame with no way to distinguish
"this is the window's initial position" from "force it back here." Real ImGui solves this with
`ImGuiCond` (Once / FirstUseEver / Always / Appearing) and window state that is **owned by the
library, not the caller**. v2 never mentions this, so its flagship code sample demonstrates a
window that cannot actually be moved. v3 fixes this (§6.8).

**Structural gaps** (present in the ImGui fidelity audit, absent from v2):
- No description of the **context object** — where hovered/active/focused IDs, stacks, and
  per-window state actually live between frames.
- No **frame lifecycle** — v2 says widgets should behave immediate-mode but never specifies
  *when* layout, hit-testing, interaction, and drawing happen relative to each other, which is
  exactly how "reacts one frame late" bugs get built.
- No **`PushItemWidth`** — without it, every slider/drag/input call in real ImGui code would need
  an explicit pixel width, which real ImGui code never does. v2's own API examples
  (`im.slider('Volume', 0, 100, app.volume)`) have no width argument and no mechanism to supply one.
- No **last-item query API** (`IsItemHovered()`/`IsItemActive()`/`IsItemEdited()`). This is the
  primitive that real ImGui tooltips, drag-drop sources, and right-click context menus are built
  on. Without it, "attach a tooltip to whatever was drawn a moment ago" has no mechanism.
- No **`BeginGroup`/`EndGroup`**, no window **flags**, no distinction between **table sorting
  being application-owned** vs. library-owned, no **nested-popup-stack closing rule**, no
  **virtualization** requirement for large tables, no **DPI**, **security**, or **state
  persistence** sections, and no **priority levels** or **phased roadmap** — all requested by the
  review brief and all necessary for an agent to build this without guessing.
- One factual overreach: v2 §2's CSS (`font-smooth: never; image-rendering: pixelated;`) does not
  achieve the bitmap-pixelation effect it's presented as producing — `image-rendering` only
  affects raster/canvas scaling, not vector font glyph rendering, and `font-smooth` is
  non-standard and ignored by current Chromium. Corrected in §6.3.
- Real ImGui's own **Columns()** API is legacy/soft-deprecated in favor of Tables; v2 doesn't
  mention Columns at all, which is actually correct — v3 makes that omission an explicit decision
  rather than a silent one, so nobody re-adds it later.
- **Docking** (drag-a-tab-to-snap-into-another-panel) is a distinct branch of real ImGui, not
  core Dear ImGui. v2's Splitters section gestures at "dockable-feeling" panels without saying
  docking itself is out of scope. v3 states this as an explicit non-goal.

None of v2's visual/token work is wrong — it's kept, condensed, and used as the base layer of v3.

---

## 2. Major Changes

1. **Added a real architecture layer beneath the widget list**: Context, Frame Lifecycle, ID
   System, Layout Engine, Input System, and the Hover/Active/Focus state machine are now first-class
   sections with implementation-level detail, not implied by a widget catalog.
2. **Fixed the window-position contradiction** by specifying `ImGuiCond`-equivalent semantics and
   making window position/size persisted-and-owned rather than re-asserted every frame.
3. **Added the widget contract** — a single template every widget module must satisfy (ID, rect,
   size, interaction state, render state, a11y info, keyboard behavior, mouse behavior, persistent
   state, return value) so widgets don't each reinvent hit-testing and state handling.
4. **Split "popup" into its actual four distinct behaviors** (popup, modal, tooltip, context-menu-on-item)
   with a shared open/close/z-order model instead of ad hoc per-widget rules, and added the
   nested-popup-stack closing rule real ImGui uses.
5. **Made table sorting explicitly application-owned** (the widget reports sort-spec changes; it
   never reorders the caller's array itself) and added a virtualization requirement for large
   tables/lists, which v2's performance goals (500+ widgets at 60fps) cannot be met without.
6. **Added Priority tags (P0–P3)** to every requirement and a **dependency-ordered phase plan**,
   so "what do I build first" has one answer instead of being inferred from document order.
7. **Corrected the pixel-font CSS claim** and reframed typography as two honest options (real
   bitmap font vs. vector approximation) rather than one CSS snippet presented as achieving both.
8. **Added sections v2 entirely lacked**: DPI/high-DPI, security (XSS via user-controlled labels),
   state persistence keys, error/diagnostic requirements for dev builds, a documentation
   checklist, and explicit rules for how a coding agent should operate on this codebase.
9. **Scoped out** legacy `Columns()` and true multi-window docking, explicitly, so they aren't
   silently rebuilt later under the assumption they were missed.

---

## 3. Important Requirements Added

- `PushItemWidth()` / `PopItemWidth()` / `CalcItemWidth()` — layout-scoped default widget width.
- `BeginGroup()` / `EndGroup()` — composite-item grouping for layout and hover/tooltip purposes.
- `IsItemHovered()`, `IsItemActive()`, `IsItemFocused()`, `IsItemClicked()`, `IsItemEdited()`,
  `GetItemRect()` — last-item query surface.
- `io.wantCaptureMouse` / `io.wantCaptureKeyboard` — output flags so a host page/app embedding
  this UI knows whether to swallow the underlying input event.
- Window **flags** (`NoTitleBar`, `NoResize`, `NoMove`, `NoScrollbar`, `NoCollapse`,
  `AlwaysAutoResize`, `NoBackground`, `NoSavedSettings`, `NoFocusOnAppearing`,
  `NoBringToFrontOnFocus`, `UnsavedDocument`) and a `Cond` argument for position/size.
- Explicit **ID collision and reorder-instability** documentation: index-based `PushID(i)` inside a
  reorderable/deletable list causes a *specific, real bug class* (a text field's cursor/selection
  state "jumps" to the wrong row after a delete because the row's identity, not its data, owns the
  state) — with the required fix (key by stable item identity, not array index).
- Disabled-state semantics: `BeginDisabled()/EndDisabled()` — a disabled widget still renders and
  still returns its last value, is excluded from Hovered/Active/Focus eligibility and tab order,
  and gets `aria-disabled="true"`.
- Nested popup-stack closing rule: closing only unwinds from the deepest open popup inward; a
  click inside an ancestor popup/menu does not close it; a click fully outside the whole stack
  closes all of it.
- Table sort state is **application-owned**: the widget exposes a sort-spec (column + direction +
  multi-sort order via Shift+click) and a "dirty this frame" flag; it never mutates or reorders the
  caller's data array.
- Virtualization requirement: table/list bodies beyond ~100 rows must not mount one DOM node per
  row; only render the visible window plus overscan.
- Viewport-edge clamping/flipping for tooltips, popups, combos, and submenus so they cannot render
  off-screen.
- DPI section: behavior must be verified at 100/125/150/200% OS scaling; text-metric caching must
  key on `devicePixelRatio` to avoid blurry/misaligned pixel-font rendering.
- Security section: all user-supplied labels/text go through `textContent`/DOM text nodes, never
  `innerHTML`; drag-drop payloads and table cell content are data, never interpreted as markup.
- Dev-mode diagnostics: unbalanced `Push*/Pop*` stacks, `Begin` without matching `End`, duplicate
  IDs in the same scope, `NaN`/inverted min>max ranges — all warn to console in dev builds, never
  throw in production builds.
- A **Priority legend (P0–P3)** and a **dependency-ordered phase roadmap** (§7).
- Explicit non-goals: legacy `Columns()`, true OS-level multi-viewport/docking.

---

## 4. Requirements Removed or Simplified

- **v2's literal pixel-font CSS snippet** (`font-smooth: never; image-rendering: pixelated;`) is
  removed as a claim of fact and replaced with two honestly-scoped options (§6.3) — keeping it
  as written would have shipped a font that looks like a normal sans-serif while the spec claimed
  it looked pixelated.
- **The "magenta" bonus theme** is kept only as a one-line footnote under theming (§6.13) instead
  of a fully speculated color/blur treatment — it doesn't affect fidelity or architecture, so
  giving it the same weight as the authentic/stylized presets would dilute what actually matters.
- **Plots and Splitters** are explicitly marked P2/P3 with minimal specs, not full behavioral
  contracts — they're real ImGui widgets but are not load-bearing for "does this feel like ImGui,"
  and over-speccing them would cost roadmap time better spent on tables/popups/text-input, which are.
- **Legacy `Columns()`** is dropped entirely rather than specified alongside Tables — real ImGui
  itself steers users to Tables now; cloning both is duplicate work for zero additional fidelity.
- Redundant restatement of the full color token table under both the "authentic" and "stylized"
  presets is collapsed into one table with a diff column (§6.2) — v2 printed two nearly-parallel
  tables' worth of the same nine tokens.
- v2's repeated meta-instructions ("don't make it unnecessarily complicated," "ship it as a
  toggle") are folded into the relevant requirement itself rather than kept as separate prose
  asides — they're now just part of how each requirement is worded.

---

## 5. Architecture Decisions

| Decision | Choice | Why |
|---|---|---|
| Rendering surface | **DOM + CSS**, not Canvas/WebGL | Widgets are native-feeling, ARIA/focus/text-editing come nearly free, and 60fps with hundreds of widgets is achievable with DOM node pooling. Canvas would mean reimplementing text layout, IME, selection, and accessibility from scratch for no fidelity gain — v2's own §12 already conceded native-ImGui's GPU batching model can't be matched anyway. |
| Canvas/SVG use | **Only** for `PlotLines`/`PlotHistogram` sparklines and the drag-and-drop floating ghost preview | These are the only pieces where a lightweight immediate-draw surface is actually simpler than DOM nodes; everything else stays DOM. |
| Text input | **Native `<input>`/`<textarea>`**, styled to match tokens | Free IME, caret, selection, clipboard, and OS-level a11y. Documented as an accepted deviation from ImGui's own hand-rolled STB-textedit caret (§10) rather than something to fight. |
| Layout engine | **Self-computed sequential cursor**, not CSS flexbox/grid, with widths sourced from a **cached `canvas.measureText()` font-metrics table**, not synchronous DOM `getBoundingClientRect()` per widget | A true immediate-mode layout cursor needs to know a widget's size *before* placing the next one, in the same synchronous call. Querying real DOM layout per widget would force a browser reflow on every widget call (layout thrashing) and reintroduces exactly the "reacts one frame late" risk the frame lifecycle is designed to prevent. |
| ID hashing | A fast string hash (e.g. FNV-1a) over the ID-stack-scoped string, **not** a bit-for-bit port of ImGui's CRC32 | The requirement is determinism and low collision probability, not byte-identical hash values with the C++ library — nothing external ever needs to match ImGui's literal hash output. |
| State storage | A single `Map<hashedId, WidgetState>` in the context, not state living on retained JS widget objects | Matches real ImGui's model (state keyed by ID, not object identity) and is what makes the ID-collision/reorder bug class (§3) representable and testable at all. |
| Table/list virtualization | Render only the visible row window + a small overscan buffer, recycling DOM row nodes | Required to hit the 500+ widget / large-table performance targets without literally mounting thousands of DOM nodes. |
| Retained wrapper (v2 §11's builder API) | Kept, but redefined as a **thin adapter that itself runs the immediate-mode draw function every frame internally** and diffs the boolean return values to fire callbacks | v2 presented it as "a thin adapter... not a second parallel engine" without saying how — a callback-based builder API and a per-frame return-value API don't compose unless the adapter is the one holding the `requestAnimationFrame` loop. Spelled out in §6.25. |

---

## 6. Revised Complete Specification (v4)

### 6.0 Scope and Non-Goals

**Goal:** a browser UI system that is both visually and *behaviorally* faithful to Dear ImGui —
immediate-mode call semantics, ID-stack-based state, the hover/active/focus model, and ImGui's
specific widget conventions (drag-scrub numeric fields, app-owned table sorting, instant-by-default
popups).

**Explicit non-goals** (state these in the README so "clone" isn't read as "100% of ImGui"):
- **Legacy `Columns()`/`NextColumn()`** — superseded by Tables (§6.14.12) in real ImGui; not built.
- **True docking** (drag a tab out, snap panels together via drop-zone overlays) — this is
  ImGui's separate Docking branch, not the core library. Splitters (§6.14.19) are the supported
  fixed-multi-pane alternative.
- **Multi-viewport / separate OS windows** — requires an Electron/Tauri-style shell; a browser tab
  cannot spawn independent OS windows.
- **Byte-identical GPU rendering** — see §10.

### 6.1 Priority Legend

- **P0** — Core. The clone is not "a Dear ImGui clone" without this.
- **P1** — Important. Expected in any serious ImGui-derived tool UI; ship after P0.
- **P2** — Optional. Real ImGui feature, genuinely useful, not load-bearing for fidelity.
- **P3** — Stretch. Nice, not expected, do last if at all.

### 6.2 Visual Design Tokens `[P0]`

Single source of truth, diffed across the two shipped presets (values from `ImGuiStyle::StyleColorsDark()`):

| Token | Authentic (real ImGui default) | Stylized (v1/v2 original) |
|---|---|---|
| `--color-text-primary` | `#FFFFFF` | `#e0e0e0` |
| `--color-text-secondary` (TextDisabled) | `#808080` | `#a0a0a0` |
| `--color-bg` (WindowBg) | `#0F0F0F` @ 94% | `#1e1e1e` @ 100% |
| `--color-panel` (PopupBg) | `#141414` @ 94% | `#2d2d2d` |
| `--color-title-bg` / `-active` | `#0A0A0A` / `#294A7A` | `#1a1a1a` / `#1a1a1a` |
| `--color-menu-bar` | `#242424` | `#1a1a1a` |
| `--color-frame-bg` (input idle/hover/active) | `#294A7A`@54% / `#4296FA`@40% / `#4296FA`@67% | `#1a1a1a` / `#4d4d4d` / `#5d5d5d` |
| `--color-button` (idle/hover/active) | `#4296FA`@40% / `#4296FA`@100% / `#0F87FA` | `#3d3d3d` / `#4d4d4d` / `#5d5d5d` |
| `--color-accent` (CheckMark/SliderGrabActive) | `#4296FA` | `#0088ff` |
| `--color-header` (Selectable/selected row @31/80/100%) | `#4296FA` | `#0088ff` |
| `--color-border` | `#6E6E80` @ 50% | `#454545` @ 100% |
| `--color-scrollbar-bg` / `-grab` | `#050505`@53% / `#4F4F4F` | `#1a1a1a` / `#4d4d4d` |
| `--color-plot-lines` / `-histogram` | `#9C9C9C` / `#E5B300` | — (n/a) |
| `--border-width-frame` | `0px` (fill-only state changes) | `1px solid` |
| `--radius-window` / `-frame` | `0px` / `0px` | `4px` / `2px` |
| `--radius-scrollbar` / `-tab` | `9px` / `4px` | same |
| Spacing (`WindowPadding` 8/8, `FramePadding` 4/3, `ItemSpacing` 8/4, `ItemInnerSpacing` 4/4, `IndentSpacing` 21, `ScrollbarSize` 14, `GrabMinSize` 10) | identical in both presets | identical in both presets |

**Rule:** switching `data-imgui-theme` must change zero JS and zero widget markup — only these
custom properties. This is a P0 acceptance test (§8), not a suggestion.

*Footnote — optional third preset:* a `magenta` preset (`--color-accent: #ff2e88` plus
`backdrop-filter: blur(8px)` on Selectable/table rows) is a valid additive preset once the token
system above is proven to decouple theme from markup. `[P3]`

### 6.3 Typography `[P0 baseline, P2 bitmap-accurate mode]`

Real Dear ImGui's default is `ProggyClean.ttf`, a **bitmap** font baked at exactly 13px with no
antialiasing. Most shipped ImGui apps replace it with a normal TTF (Inter/Segoe UI/etc.) via
`ImFontAtlas` — so "looks like ImGui" legitimately has two honest targets. Ship both, correctly:

- **`--font-imgui-custom`** (default, P0): `"Segoe UI", "Inter", -apple-system, sans-serif` at
  13px, weight 400. This is what most real-world ImGui apps look like.
- **`--font-imgui-classic`** (P2, corrected from v2): a genuine bitmap-look result requires either
  (a) an actual pre-rasterized bitmap font asset (a sprite-sheet-based `@font-face` or a canvas
  glyph atlas), or (b) accepting a vector "pixel-style" webfont (e.g. a Press-Start-2P-style face)
  as an *approximation*, clearly labeled as such. **Do not rely on `image-rendering: pixelated`**
  (affects raster/canvas image scaling only, not vector glyph rendering) **or `font-smooth: never`**
  (non-standard, ignored by current Chromium) to produce this effect — neither does what v2 claimed.

No real bold weight exists in ProggyClean; simulate "bold-looking" headers (`SeparatorText`) via
size/letter-spacing, not `font-weight: 700`, if targeting authentic mode.

### 6.4 Core Architecture: The Context `[P0]`

Everything below lives in one long-lived `ImGuiContext`-equivalent object, created once, passed
implicitly to every widget call (module-level singleton is acceptable for a single-instance UI):

| Field | Purpose |
|---|---|
| `windows: Map<id, WindowState>` | Persisted per-window state: position, size, scroll offset, collapsed flag, z-order, open/closed. |
| `windowStack` | Currently-open `beginWindow`/`endWindow` scopes *this frame* — used to validate nesting and catch `End()` without `Begin()`. |
| `idStack` | Hash values; top of stack = current ID scope (§6.6). |
| `styleColorStack` / `styleVarStack` / `itemWidthStack` / `fontStack` | Temporary override stacks, pushed/popped in pairs, checked balanced at end of frame (§6.13). |
| `hoveredId`, `hoveredIdPreviousFrame` | The one widget under the pointer this frame (topmost, popup/clip-aware); previous-frame value lets hover-enter/leave transitions be detected without lag. |
| `activeId`, `activeIdWindow`, `activeIdSource` | The one widget currently capturing interaction (`mouse` or `keyboard` sourced — affects release rules), and which window it belongs to (for scroll-follow / raise-to-front). |
| `focusedWindowId`, `navId` | Window-level and widget-level keyboard focus, independent of hover. |
| `lastItemData: {id, rect, hovered, active, edited, activated, focused}` | The query surface behind `IsItemHovered()` etc. (§6.9, §6.25) — set by whatever widget call ran most recently. |
| `openPopupStack`, `modalStack` | Currently-open popups in open-order, each tagged with its spawning item/window and screen rect; modal entries block hit-testing behind them. |
| `tooltipState` | Pending/shown tooltip and its hover-delay timer. |
| `dragDropState` | Active payload type/data, source id, current hover target id. |
| `currentTable`, `currentTabBar` | Valid only between `beginTable`/`endTable` and `beginTabBar`/`endTabBar`; hold column/sort/scroll-freeze state or tab-order/active-tab state. |
| `frameCount`, `time`, `deltaTime` | Advanced in `newFrame()`; drive tooltip delay and double-click timing. |
| `io` | This frame's input snapshot (§6.9) plus output flags `wantCaptureMouse`/`wantCaptureKeyboard`. |
| `style` | Resolved token values for the active preset; mutable at runtime by the Style Editor (§6.14.20). |
| `persistedState` | Cross-frame/cross-reload store keyed by ID: window pos/size/collapsed, tree-node open state, active tab, table column widths/sort (§6.22). |

### 6.5 Frame Lifecycle `[P0]`

1. **Input collection** (continuous, before `newFrame()`): a single input manager accumulates
   `pointerdown/move/up/cancel`, `wheel`, `keydown/up`, and `resize` events into a pending buffer.
   No context state changes here.
2. **`newFrame()`**: snapshot the pending buffer into `io` and clear it; advance `frameCount`,
   `time`, `deltaTime` (from the rAF timestamp, not `Date.now()` polling); copy `hoveredId` →
   `hoveredIdPreviousFrame` then clear `hoveredId` (recomputed fresh this frame); clear
   `lastItemData`; if a mouse-up occurred with no matching capture, clear `activeId`.
3. **Application draw callback** runs (`drawUI(app)` — the app's own function, called once). Inside
   it, every `Begin*`/widget/`End*` call performs, **synchronously, in that one function call**:
   - **3a. Layout** — request size from cached font metrics + style tokens; advance the cursor.
   - **3b. Hit test** — compare the widget's rect to `io.mousePos`, the active clip-rect stack, and
     open-popup/modal boundaries; claim `hoveredId` only if nothing has already claimed it this
     frame at a higher z-order.
   - **3c. Interaction** — using hover + previous `activeId` + this frame's mouse button edges,
     run the widget's own state machine (§6.9) and compute its return value.
   - **3d. Draw** — update/create the widget's pooled DOM node(s) to reflect *this frame's* state,
     immediately, in the same call — never deferred to a later "diff" pass.

   **The rule that prevents one-frame-lag bugs:** a widget's hit test in step 3b must use the rect
   *this same call* computed in 3a — never a rect cached from last frame's DOM measurement.
4. **Hit-test finalization**: if nothing claimed `hoveredId` this frame, it stays `null` (pointer
   over empty space).
5. **Interaction post-processing**: click-outside-closes-popup logic runs here, by checking whether
   this frame's `pointerdown` target rect matched *any* widget/popup rect drawn this frame.
6. **DOM sync**: the pooled-node reconciler moves/reuses nodes to match this frame's declared
   widget order; only nodes whose state actually changed are touched.
7. **Post-frame persistence**: state flagged dirty this frame (window moved/resized, tree toggled,
   tab switched, column resized) is written to `persistedState`, coalesced rather than written
   every intermediate frame of a drag.
8. **`render()`**: finalizes the frame. For a DOM backend this can be a no-op merged into step 6;
   kept as an explicit call so a future canvas/WebGL backend could be swapped in without changing
   application code.
9. **`requestAnimationFrame`** schedules the next iteration.

### 6.6 ID System `[P0]`

- Every widget has an **ID** derived from its label plus the current `idStack`, computed via a
  fast, deterministic string hash (FNV-1a or equivalent) — bit-for-bit parity with ImGui's own
  CRC32 is **not** a requirement; determinism and low collision probability are.
- `PushID(str | int | ref)` / `PopID()` push/pop an explicit scope, used to disambiguate loops.
- **`##`**: text after a `##` is not *displayed* but **is** included in the hash — `"Delete##row1"`
  shows "Delete" but hashes the full string, so two buttons both labeled "Delete" with different
  `##` suffixes get different IDs.
- **`###`**: only the text **after** `###` contributes to the hash; text before it still displays.
  Use this to keep a widget's identity/state stable while its visible label changes
  (`"Saving…###status"` → `"Saved###status"` is one widget across frames, not two).
- **Window/child/popup/table/tab IDs** are just widget IDs in their own right, pushed onto the
  stack for the duration of their content, which is what automatically namespaces widget IDs
  inside a child window away from same-named widgets in the parent.
- **Duplicate-ID detection**: in dev builds, warn to console (never throw) if two items resolve to
  the same ID within the same scope in the same frame.
- **The reorder/deletion bug class (must document, must test):** `PushID(index)` inside a
  reorderable or deletable list gives item *state* (a text field's cursor position, a tree node's
  open/closed flag) to a **position**, not to the underlying data. Delete row 2 of 5, and row 3's
  state is now attached to what displays as row 2 — a text cursor appears to "jump" to the wrong
  item. **Required fix:** key `PushID` by a stable identity from the data itself (a database id, a
  UUID, an object reference) whenever the list can be reordered or have items removed; index-based
  IDs are only safe for lists that are fixed for the widget's lifetime. This must be called out in
  the docs and covered by an explicit regression test (§8).

### 6.7 Layout Engine `[P0]`

- **Cursor**: `{x, y}` relative to the current window's content region; advances downward by
  default after each item by that item's height + `ItemSpacing.y`.
- **`SameLine(offsetX?, spacing?)`**: cancels the pending line break, placing the next item at the
  previous item's right edge + spacing (or at an explicit `offsetX` from the line's start).
- **`NewLine()`**, **`Spacing()`**, **`Dummy(w, h)`** (reserves an empty layout rect — for manual
  gaps or as a drag-drop/custom-draw anchor).
- **`Indent(px?)` / `Unindent(px?)`**: shifts the content region's left edge by `IndentSpacing`
  (default) or an explicit amount; used by `TreeNode` nesting.
- **`BeginGroup()` / `EndGroup()`** *(added, missing from v2)*: everything drawn between the two
  calls is collapsed, for the outer cursor and for `IsItemHovered()`/`IsItemActive()` immediately
  after `EndGroup()`, into one bounding rect — needed to build a label+widget row that behaves (and
  is tooltippable) as a single unit, and to `SameLine()` after a multi-line block.
- **`PushItemWidth(px)` / `PopItemWidth()` / `CalcItemWidth()`** *(added, missing from v2)*:
  controls the width the **next** value-widget(s) request, independent of label length. Positive =
  fixed pixels. Negative = "content-region right edge minus `|px|`" (lets a widget's right edge
  track a fixed margin across window resizes). Unset = engine default of `0.65 × availWidth`.
  Without this, every slider/drag/input/combo call site would need an explicit width argument,
  which real ImGui code never does — and v2's own API examples had no mechanism to supply one.
- **Text measurement**: widths for label-bearing widgets come from a **font-metrics cache** built
  once per `(font, size, devicePixelRatio)` via an offscreen `canvas.measureText()` pass — **never**
  from reading back a real DOM node's `getBoundingClientRect()` mid-frame. The latter forces a
  synchronous browser reflow on every widget call (layout thrashing) and is exactly the kind of
  per-widget DOM query that reintroduces one-frame-lag risk.
- **Resizing**: when a window resizes, the *next* frame's layout pass simply recomputes
  `availWidth` from the new rect — there is no retained layout tree to invalidate, because immediate
  mode never had one.
- **Explicitly out of scope**: legacy `Columns()`/`NextColumn()` — superseded by Tables (§6.14.12)
  in real ImGui itself; building both would duplicate effort for no fidelity gain. `[non-goal]`

### 6.8 Window System `[P0]`

**Flags** (bitmask or option-object, applied at `beginWindow`): `NoTitleBar`, `NoResize`, `NoMove`,
`NoScrollbar`, `NoScrollWithMouse`, `NoCollapse`, `AlwaysAutoResize`, `NoBackground`,
`NoSavedSettings` (exclude this window from persistence), `NoFocusOnAppearing`,
`NoBringToFrontOnFocus`, `UnsavedDocument` (shows a dot; shared with Tab items).

**Position/size ownership — the fix for the v2 contradiction:** window position and size are
**owned by the library's persisted state**, keyed by window ID, not re-derived from whatever the
caller passes each frame. `beginWindow(id, { x, y, w, h, cond })` takes a `cond` argument mirroring
`ImGuiCond`:
- `Once` *(default)* — apply the supplied position/size only once during the lifetime of the current
  ImGui context. After that, user interaction owns the value.
- `FirstUseEver` — apply the supplied position/size only when no persisted settings exist for this
  window ID. Existing persisted settings always win.
- `Appearing` — apply the supplied position/size when a previously hidden/closed window transitions
  to visible.
- `Always` — apply the supplied position/size every frame, overriding any user movement and resizing;
  rarely wanted for pos/size, useful for programmatic layout resets.

**Default `beginWindow(name, options?)` contract**:
- `x`: `undefined` by default; use persisted state if available, otherwise library default.
- `y`: `undefined` by default; use persisted state if available, otherwise library default.
- `w`: `400` by default.
- `h`: `300` by default.
- `cond`: `ImGuiCond.Once` by default.
- `flags`: `0` by default.

`w`/`h` are treated as initial size values under `Once`/`FirstUseEver`/`Appearing`, not as a permanent
force-set value. User resizing continues to own the value after the initial seed.

With this, calling `beginWindow('Settings', {x:20, y:20, w:300})` every frame — exactly v2's own
example — now only **seeds** the position once; user drags persist correctly thereafter.

- **Dragging the title bar**: `pointerdown` in the title-bar rect (not on the collapse arrow or
  close button) claims `activeId` as a synthetic `window-move:{id}`; drag delta updates persisted
  position every frame while active; the final position is written to `persistedState` once, on
  release (not on every intermediate frame — see Performance §6.20).
- **Resizing**: 8px edge grab margin, 12px corner grab zone (cursor becomes the matching
  `*-resize`); clamps to a configurable min-size (default 100×60) and to the viewport if
  configured; entirely disabled if `NoResize` is set.
- **Z-order/focus**: clicking anywhere inside a window's rect (title bar, body, or any child
  widget) raises it and sets `focusedWindowId`, unless `NoBringToFrontOnFocus` is set. Exactly one
  window is `focusedWindowId` globally. Losing focus does not reset scroll/collapsed state.
- **Collapse**: the ▼/▶ arrow toggles a persisted boolean; when collapsed, only the title bar
  renders, but `beginWindow`/`endWindow` must still be called in a balanced pair — mirror ImGui's
  own idiom: `if (!im.beginWindow(...)) { im.endWindow(); return; }` lets the app skip building the
  body's widgets while keeping the call balanced.

  **`beginWindow()` return contract**:
  - `true` = the window is open and its contents should be submitted this frame.
  - `false` = the window exists but its contents should not be submitted this frame because it is
    collapsed, hidden, or otherwise suppressed.
  - Even when `false` is returned, the matching `endWindow()` call is mandatory.
  - **Closed** and **collapsed** are distinct states: a closed window is not drawn and is not part
    of the current frame's widget submission; a collapsed window is still part of the current frame,
    but only its title bar renders.
- **Child windows**: `beginChild(strId, size, border?, flags?)` opens an independent scroll region
  that also pushes its own **ID scope**, automatically namespacing widget IDs inside it away from
  the parent window and sibling children (ties to §6.6).
- **Docking is explicitly out of scope** (§6.0) — Splitters (§6.14.19) are the supported
  fixed-multi-pane alternative.

### 6.9 Input System `[P0]`

A single input manager owns all listeners — **no widget installs its own event listener directly.**

- Pointer events (`pointerdown/move/up/cancel`) preferred over mouse events, for unified
  touch/mouse handling and pointer capture (§6.11).
- `io`: `mousePos`, `mouseDown[3]`, derived edges `mouseClicked[]`/`mouseReleased[]` (computed once
  per `newFrame()` by comparing this frame's `mouseDown` to last frame's — **not** re-derived
  redundantly inside every widget), `mouseDoubleClicked[]` (timing-windowed), `wheelDelta`,
  `keysDown` set, active modifiers, and text-input events for native inputs.
- **Output flags** `io.wantCaptureMouse` / `io.wantCaptureKeyboard`: true whenever `hoveredId` or
  `activeId` is non-null, or a native input has focus — so a host page/app embedding this UI over
  other content (a canvas game, say) knows whether to `stopPropagation`/suppress its own handling
  of that input event. *(Missing from v2 entirely; required for this to be embeddable at all.)*
- Query API mirrors real ImGui naming where it's genuinely useful: `IsMouseDown(btn)`,
  `IsMouseClicked(btn)` (true only on the down-edge, not while held), `IsMouseReleased(btn)`,
  `IsMouseDoubleClicked(btn)`.
- **Pointer IDs**: the pointer that claims `activeId` becomes `activePointerId`; other pointer events
  cannot modify the active interaction. `pointerup` / `pointercancel` only release `activeId` when
  `pointerId === activePointerId`. This is required for deterministic multi-pointer behavior.
- **Browser-event conversion**: the input manager must explicitly handle `pointerdown`/`pointermove`/
  `pointerup`/`pointercancel`, `wheel`, `keydown`/`keyup`, `input`, `compositionstart`/
  `compositionupdate`/`compositionend`, `paste`, `contextmenu`, `blur`, `visibilitychange`, and
  `resize`. Blur and visibility changes while dragging must cancel or release the interaction
  deterministically.

### 6.10 Interaction State Machine `[P0]`

| State | Definition | Cardinality | Notes |
|---|---|---|---|
| **Hovered** | Pointer inside the widget's rect **and** it is the topmost hoverable thing there (a widget under an open popup/modal is never Hovered even if geometrically overlapped). | At most one, globally | Recomputed fresh every frame in step 3b of the lifecycle. |
| **Active** | Currently capturing interaction — mouse held on it, a drag/slider being scrubbed, a text field placing its caret. | At most one active interaction exists globally | **Persists even if the pointer leaves the widget's rect** (§6.11) until pointer-up; tagged with its capture source (`mouse`/`keyboard`), which affects release rules. |
| **Focused** | Keyboard focus, independent of the mouse. | At most one keyboard-focus target exists globally | `focusedWindowId` identifies the focused window; `navId` identifies the focused keyboard-navigable item within that window. A widget can be focused via Tab without being Hovered. |

**Hover resolution rule**: hover is not determined solely by declaration order. Each submitted item receives a z-layer, window z-order, popup depth, and declaration sequence. After submission, the hit-test finalizer selects the highest-priority hoverable item under the pointer. Priority is: modal > popup/submenu > focused/normal window > background window; within a layer, higher window z-order wins. An earlier-submitted widget cannot steal hover from a later/higher window solely because it was declared earlier.
| **Disabled** | Inside a `BeginDisabled()/EndDisabled()` scope. | Per-widget | Still renders, still returns its unchanged last value; excluded from Hovered/Active/Focus eligibility and tab order; gets `aria-disabled="true"`. |
| **Pressed / Edited / Activated** | Per-frame transient flags on `lastItemData`, not persistent states. | Per-widget, per-frame | `Activated` = became true this exact frame; `Edited` = value changed this exact frame. |
| **Selected / Opened / Dragging** | Persistent, widget-family-specific state (Selectable's selected row, TreeNode's open flag, an active drag-drop source). | Per-widget | Lives in `persistedState` or the relevant sub-state (table/drag-drop), not in the generic hover/active machine. |

**Button click rule** (the detail naive `onclick` implementations get wrong): a click fires only
when `pointerdown` occurred over the widget (→ becomes Active) **and** `pointerup` occurs while the
widget is *still both Hovered and Active*. Dragging off before release clears Active on release and
fires no click — this "forgiving precision" / drag-out-to-cancel behavior is required, not
incidental (see §8 for its explicit test).

**Focus and active capture are independent**: if `activeId` is owned by a mouse drag, changing
keyboard focus does **not** cancel the active mouse interaction. Pointer capture remains authoritative
until `pointerup` or `pointercancel`.

If a widget loses keyboard focus while it has keyboard-sourced active state, the widget-specific
keyboard interaction is cancelled unless the widget explicitly defines otherwise.

If the owner of `activeId` is removed, closed, or destroyed while active:
- release pointer capture
- clear `activeId`
- clear `activePointerId`
- cancel the interaction
- do not emit `Activated`/`Edited` for the cancelled interaction

At most one active interaction exists globally and at most one keyboard-focus target exists globally.

**DOM focus ownership rule**: `document.activeElement` is an implementation detail, not the source of truth. The ImGui context owns logical keyboard focus. When a native input receives focus, the context updates `focusedWindowId` and `navId` as needed. When logical focus changes, the implementation synchronizes DOM focus as required. A DOM focus change must never silently change ImGui state without passing through the input/focus manager.

**Last-item queries** *(added, missing from v2)*: after any widget call, `IsItemHovered()`,
`IsItemActive()`, `IsItemFocused()`, `IsItemClicked()`, `IsItemEdited()`, `GetItemRect()` read
`lastItemData` from the call that just ran. This is the primitive real ImGui tooltips
(`if (IsItemHovered()) SetTooltip(...)`), drag-drop sources, and item-attached context menus are
built on — without it there is no way to say "attach X to whatever was just drawn."

### 6.11 Pointer Capture `[P0]`

- On `pointerdown` that claims `activeId`, call `element.setPointerCapture(pointerId)` on the
  widget's DOM node so subsequent `pointermove`/`pointerup` keep firing on it even outside its own
  box, the containing window, or the browser viewport, subject to browser and OS limitations.
  **Pointer capture must continue delivering pointer events while the pointer leaves the widget,
  containing window, or browser viewport, but it cannot guarantee event delivery after the pointer
  leaves the browser application's OS-level window boundary.** This is documented as an
  unavoidable limitation, §10.
- Release capture on `pointerup` or `pointercancel`; this is also the only point at which
  `activeId` is cleared for a mouse-sourced capture (keyboard-sourced capture, e.g. a nav-selected
  slider, releases on the relevant key-up/Escape instead).
- **Required test sequence**: `pointerdown` on a drag/slider → move past the widget's rect → move
  past the window's rect → `pointerup` far outside both. The value must have kept updating
  correctly through every intermediate move and stopped exactly at release.

### 6.12 Widget Contract `[P0]`

Every widget module implements the same ten-part contract, backed by shared primitives so this
logic exists once, not once-per-widget:

1. **ID** (§6.6) 2. **Bounding rect** (from layout) 3. **Layout size** (from font-metrics cache +
style tokens) 4. **Interaction state** (§6.10, via shared `hitTest()`/`claimHover()`/`claimActive()`/
`releaseActive()`) 5. **Render state** (pooled DOM node reflecting this frame) 6. **Accessibility
info** (role/label/`aria-*`, via a shared `pushA11y()` helper) 7. **Keyboard behavior** (own
handler, but focus/tab-order bookkeeping is shared) 8. **Mouse behavior** (via shared pointer-capture
helpers, §6.11) 9. **Persistent state**, if any, read/written through `persistedState` (§6.22) 10.
**Return value** — see the contract split below.

**Return-value convention**: *value widgets* (`slider`, `drag`, `inputText`, `checkbox`, `combo`)
return the (possibly updated) value directly — a functional style, since JS has no pointer-out
parameters. The value returned is the current value for the widget on that frame; `IsItemEdited()`
reports whether the value changed during the current frame. *Action widgets* (`button`, `menuItem`,
`selectable`-as-click) return `true` only on the exact frame the action completed.

**Examples**:
- `inputText(label, value, options?)` returns the current string every frame; `IsItemEdited()` is true
  only when the string changed during this frame; `Escape` restores the value captured when the edit
  session began.
- `slider(label, min, max, value, options?)` returns the current numeric value every frame; the
  implementation clamps it to the allowed range and `IsItemEdited()` indicates whether it changed this
  frame.
- `drag(label, value, options?)` returns the current numeric value every frame while the drag is active;
  it does not require a separate setter callback.
- `checkbox(label, value)` returns the current boolean every frame; `true` indicates checked.

### 6.13 Style/Color/Width Stacks `[P0]`

- `PushStyleColor(token, value)` / `PopStyleColor(count = 1)`; `PushStyleVar(token, value)` /
  `PopStyleVar(count = 1)`. Both stacks are checked balanced at end-of-frame in dev builds — an
  unbalanced stack warns to console (never throws) and auto-corrects by popping the remainder, so a
  caller mistake in one frame doesn't corrupt every subsequent frame's styling.
- `PushItemWidth`/`PopItemWidth` (§6.7) is the same stack pattern, kept separate because it's
  layout, not styling.
- **Theming rule (hard requirement, tested in §8)**: switching `data-imgui-theme` must not require
  touching JS or widget markup — only CSS custom properties change.

### 6.14 Widget Catalog

Appearance values for all of these come from §6.2/§6.3 and are unchanged from v2 unless noted.
Only behavioral corrections and additions are detailed below.

#### 6.14.1 Text & Labels `[P0]` — unchanged from v2.

#### 6.14.2 Buttons `[P0]`
Idle fill is the accent color at 40% alpha (not neutral gray) — buttons read "blue-tinted" even at
rest in stock ImGui. Variants: **SmallButton** (no vertical frame padding, for inline use next to
text) and **ArrowButton** (square, `▲▼◀▶` glyph, used by tree-node toggles/steppers). Click rule
per §6.10.

#### 6.14.3 Checkbox / Radio Button `[P0]` — unchanged from v2.

#### 6.14.4 Selectable `[P1]`
Full-width row, transparent when idle, `Header`-colored fill at 31/80/100% alpha across
idle/hover/selected. Click toggles/sets selection. Distinct from a `ListBox` item only in that it's
usable standalone (e.g. a clickable-but-not-part-of-a-list settings row).

#### 6.14.5 Text Input `[P0]`
Native `<input>`/`<textarea>`, styled to tokens (§Architecture Decisions). Adds: **hint text**
(`InputTextWithHint`-equivalent grayed placeholder), **step buttons** (`–`/`+` for `InputInt`/
`InputFloat`, click-and-hold repeats with acceleration), **multiline** fixed-height scrollable
variant. Filters (`CharsDecimal`, `CharsHexadecimal`, etc.) are applied on input via a validation
callback, not by trying to reproduce ImGui's C-buffer-based `CallbackResize` model — JS strings
aren't fixed buffers, and this ImGui-specific concept is correctly dropped rather than emulated.

#### 6.14.6 Drag Widgets `[P0]` — the single largest gap in the pre-v2 lineage, still needs precision
- Appearance: identical box to a slider's value display, **no visible track** — looks like a plain
  numeric field.
- Behavior: click-and-drag horizontally anywhere on the widget scrubs the value; a configurable
  pixels-per-unit `speed` controls sensitivity. Compute delta from **cumulative pointer movement
  since drag start**, not frame-to-frame deltas alone, to avoid drift/jumpiness from browser pointer
  coordinate rounding. Respect `min`/`max` clamping and a `format` string for display precision.
  **Ctrl+Click** converts it to a typeable text field for exact entry; **Escape** while typing
  cancels and restores the pre-edit value. Keyboard: Left/Right nudge by a small step once focused.
- This is how the overwhelming majority of numeric tweaking happens in real ImGui apps
  (`DragFloat`/`DragInt`) — sliders are for bounded 0–100%-style values; drag widgets are for
  everything else (position, scale, arbitrary floats).

#### 6.14.7 Sliders `[P0]`
As drag widgets, plus a visible track/thumb and value mapped from normalized `[0,1]` to
`[min,max]`. **Ctrl+Click** also converts to a typeable field (same convention as drag widgets, for
entering values outside the visual range).

#### 6.14.8 Combo / ListBox `[P1]`
The closed box shows the current selection with the same idle/hover/active fill states as a
Button, not a bordered input box — a combo *is* a button that opens a popup instead of firing an
action. Popup follows the popup open/close/viewport-clamp rules in §6.14.15.

#### 6.14.9 Color Widgets `[P1]` — three distinct things, kept distinct
**ColorButton** (bare swatch) → **ColorEdit** (swatch + inline RGBA fields, no popup — the common
settings-panel case) → **ColorPicker** (full gradient-square + hue/alpha sliders, opened by
clicking a ColorButton/ColorEdit's swatch).

#### 6.14.10 Progress Bar `[P1]`
Non-interactive; no hover/active states; `aria-valuenow` for accessibility; optional centered
percentage label overlaid on the fill.

#### 6.14.11 Tree Node / Collapsing Header `[P1]`
**CollapsingHeader** = single top-level open/closed section, no children hierarchy.
**TreeNode** = nestable, `IndentSpacing` (21px) per level, `▶`/`▼` arrow. Both toggle by clicking
anywhere on the row, not just the arrow. Open/closed state persists per §6.22, keyed by stable ID
(same reorder caveat as §6.6 applies if tree nodes are generated from a reorderable list).

#### 6.14.12 Tables `[P1]` — expanded significantly; this is a real gap area
- **Columns**: header row with labels, optional per-column sort arrow, thin vertical border between
  columns, optional alternating row background (`rgba(255,255,255,0.02)` on odd rows).
- **Sizing policies**: `Fixed` (drag to resize, min-width clamp, doesn't reflow other columns
  beyond that clamp), `Stretch` (fills remaining space proportionally), `AutoResize` (fits content
  once, no manual resize).
- **Sorting is application-owned** *(added — real and important)*: the table widget tracks and
  exposes sort specs (column, direction, and multi-sort order via Shift+click for a secondary key)
  and a "sort specs changed this frame" flag. **It never reorders the caller's data.** The
  application reads the sort-spec, re-sorts its own array, and passes the new order back in. This
  matches real ImGui's own `TableGetSortSpecs()` model and must be documented clearly enough that
  nobody implements in-widget sorting by mistake.
- **Frozen header**: the header row (and optionally leftmost columns) stays fixed while the body
  scrolls vertically; horizontal scroll (when columns overflow) moves header and body together.
- **Performance — virtualization is required, not optional**: tables/lists beyond roughly 100 rows
  must render only the visible row window plus a small overscan buffer with row nodes recycled from
  a pool, not one DOM node minted per row. This is a hard prerequisite for the 500+ widget / 60fps
  performance target (§6.20) and was entirely unaddressed in v2's table section.

#### 6.14.13 Tabs `[P1]`
Active tab, tab switching, closeable tabs (small `×`, if enabled), reorder-by-drag (if enabled),
an "unsaved" dot indicator (shared with `UnsavedDocument` window flag), overflow via `◀▶` scroll
arrows (not wrapping/shrinking tabs unreadably) rather than wrap-or-shrink, keyboard navigation
(arrow keys move between tabs when the tab bar has focus), and persisted active-tab selection
(§6.22).

#### 6.14.14 Menus `[P1]`
Menu bar → Menu → MenuItem (optionally checkbox/radio-styled with a ✓/● glyph) → Submenu (`▶`
indicator, opens on hover after a short delay, with the diagonal "safe zone" so sweeping the mouse
toward the submenu doesn't accidentally close it — a real, deliberate ImGui behavior). Keyboard:
Escape closes the current menu level; arrow keys navigate; Enter/Space activates.

#### 6.14.15 Popups, Modals, Tooltips, Context Menus `[P0]` — unified model, real gap area
Four distinct behaviors sharing one open/close/z-order mechanism (`openPopupStack`, §6.4):

| | Closes on click-outside | Closes on Esc | Blocks input behind it | Opens via |
|---|---|---|---|---|
| **Popup** | Yes | Yes | No (only its own layer) | explicit `openPopup(id)` |
| **Modal** | **No** — only an explicit action inside it, or Esc if opted in | Optional | **Yes**, everything behind it, plus a dim overlay `rgba(0.2,0.2,0.2,0.35)` | explicit `openPopup(id, {modal: true})` |
| **Tooltip** | N/A — follows hover, not click | N/A | No | automatic, after a hover delay (~0.3–0.5s, configurable), shown/hidden **instantly**, no fade |
| **Context menu on item** | Yes | Yes | No | right-click on the *last drawn item* (uses `lastItemData`, §6.10) |

**Nested popup-stack rule** *(added, missing from v2 — real ImGui behavior)*: a click closes only
from the **deepest currently-open popup inward**. A click inside an ancestor popup/menu in the
stack (e.g., clicking the parent menu while a submenu is open) does not close the ancestor; a click
fully outside the entire stack closes all of it. Opening a new popup at a shallower stack depth
than one already open closes anything deeper first. This matters for combo-inside-popup and
menu-inside-menu cases and was previously handled only as flat "close on click outside."

**Viewport clamping** *(added)*: popups, tooltips, combos, and submenus must flip/clamp their
position when they would render off the right or bottom edge of the viewport, mirroring real
ImGui's auto-flip behavior.

#### 6.14.16 Windows / Child Windows / Scrollbars `[P0]` — see §6.8 for window behavior.
Scrollbar track width 14px, thumb min-length 10px (`GrabMinSize`), matching §6.2's tokens. Mouse
wheel over a hovered child scrolls it first; only bubbles to the parent once the child hits its
scroll limit (native CSS overflow scroll-chaining is sufficient here — do not hand-roll a custom
wheel-bubbling algorithm, which is easy to get subtly wrong).

#### 6.14.17 Drag & Drop `[P2]`
A drag source shows a small floating translucent "ghost" preview following the cursor once a
threshold (~a few px of movement) is exceeded, disambiguating from a click. A valid drop target
highlights with the accent color while a compatible payload hovers over it. Implemented with
native browser drag events or a manually-positioned floating element — either way, this is
documented as visually distinct from real ImGui's own draw-list-rendered payload preview (§10).

#### 6.14.18 Plots (`PlotLines`/`PlotHistogram`) `[P2]`
Small inline sparkline/bar chart from a number array, no axes/labels by default, drawn in
`#9C9C9C` (lines) / `#E5B300` (histogram). A thin `<canvas>` or inline SVG is appropriate here —
this is the one widget family where Canvas genuinely is simpler than DOM.

#### 6.14.19 Splitters `[P3]`
A thin (4–6px) invisible-until-hovered draggable strip between two regions, proportionally resizing
both; cursor becomes `col-resize`/`row-resize` on hover. This is the supported alternative to real
docking (§6.0) for fixed multi-pane layouts — it does **not** provide drag-to-dock, floating
dock-node creation, or drop-zone overlays, and should not be described to users as "docking."

#### 6.14.20 Style Editor `[P3]`
A live-editable panel exposing every token from §6.2 with sliders/color pickers — doubles as an
internal QA tool for verifying the theming architecture actually decouples from markup.

### 6.15 Global Keyboard/Mouse Reference `[P0/P1]`

- **Ctrl+Click** on a Slider or Drag widget → typeable exact-value entry (§6.14.6/6.14.7).
- **Shift+Click** on a sortable table header → secondary sort key (§6.14.12).
- Drag threshold (~a few px) before a drag gesture registers, disambiguating from a click.
- **Tab order** follows **declaration order within the current window**, not raw DOM order. Because
  DOM nodes are pooled/reused (§6.20), `tabindex` must be **reassigned every frame** to match this
  frame's declaration order, restricted to currently-visible/non-clipped items — a stale `tabindex`
  left over from a previously-shown-then-hidden widget is a real bug class to test for explicitly.
- **Keyboard navigation** `[P1]`: Tab/Shift+Tab move focus between widgets in declaration order;
  arrow keys move within grid-like widgets (tables, tab bars) when they have focus; Space/Enter
  activates the focused widget.

### 6.16 Animation & Motion `[P0 authentic / P2 comfort]`

Stock Dear ImGui has essentially **no tweened animation** — windows, menus, popups appear/disappear
instantly; this snappiness is deliberate, not an oversight. The only real timing is the tooltip
hover-delay (§6.14.15). Ship an explicit, off-by-default toggle:

```css
:root[data-imgui-motion="authentic"] { --transition-duration: 0ms; }
:root[data-imgui-motion="comfort"]   { --transition-duration: 100ms; }
```

Default to `authentic`. `comfort` is a documented, intentional deviation for teams who find instant
popups jarring in a mouse-driven browser context — never the reverse (don't default to eased
transitions and call it authentic).

### 6.17 Accessibility `[P1 baseline, P2 high-contrast preset]`

Native ImGui has no OS-level accessibility at all (no DOM to attach it to) — a browser clone can
and should do better, but some literal-fidelity choices (40%-alpha button fill on near-black,
drag-only widgets with no visible affordance, 10px scrollbar grabs) will fail plain WCAG
contrast/target-size guidance if implemented exactly as specified. **Resolution**: keep the
Authentic visual preset as default, but layer semantic HTML/ARIA roles, visible focus rings (2px
solid accent, 2px offset), and full keyboard equivalents for every drag/scrub interaction
underneath it regardless of preset — these are structural additions, not visual ones, so they don't
compromise fidelity. Offer a separate **high-contrast token preset** (boosted fill alpha, thicker
borders) as opt-in, rather than diluting the authentic default to satisfy it.

### 6.18 DPI / High-DPI `[P1]`

Test at 100/125/150/200% OS scaling. Font-metrics caching (§6.7) must key on
`(font, size, devicePixelRatio)` — a metrics table built at 100% and reused at 150% will misalign
pixel-font layout. Any canvas-drawn elements (plots, drag-drop ghost) must be sized in device
pixels (`canvas.width = cssWidth * devicePixelRatio`) and scaled back down via CSS to avoid blur.

### 6.19 Responsiveness `[P1]`

This is a desktop/tool UI, not a responsive marketing site — do not reflow it like one. It must,
however: not break when the viewport shrinks (windows clamp to stay at least partially on-screen),
handle overlapping panels and content exceeding available space via scrolling/clipping rather than
overflow, and respect a configurable minimum window size (§6.8).

### 6.20 Rendering & Performance Architecture `[P0]`

DOM + CSS (see Architecture Decisions §5) with:
- **Node pooling**: pooled DOM nodes reused/diffed across frames, never torn down and rebuilt
  wholesale each frame.
- **No per-widget `getBoundingClientRect()` in the hot path** (§6.7) — geometry comes from the
  layout engine's own cursor math and the cached font-metrics table.
- **Event delegation** where practical, rather than one listener per pooled node.
- **Batched DOM writes** within a single frame; avoid interleaving reads and writes that would
  force synchronous layout recalculation.
- **Virtualization** for tables/lists beyond ~100 rows (§6.14.12) — mandatory, not optional, for
  the performance target below.
- **Target**: sustained 60fps with ≥500 live widgets in a single window, and with a virtualized
  table of 10,000+ rows scrolling smoothly.
- **Cleanup**: no detached event listeners or retained closures after 100 open/close cycles of a
  menu, popup, or modal — test explicitly (§8).

### 6.21 Error Handling & Dev Diagnostics `[P1]`

Dev-build-only console warnings (never thrown in production) for: duplicate IDs in the same scope
(§6.6), unbalanced `Push*/Pop*` style/width/font stacks (auto-corrected by popping the remainder),
`beginWindow`/`beginTable`/`beginPopup` without a matching `end*` call, `NaN` widget values, and
inverted (`min > max`) slider/drag ranges.

### 6.22 State Persistence `[P1]`

There are two distinct concepts that must not be conflated:

- **Runtime state**: survives across frames inside the current `ImGuiContext`; it is required for
  immediate-mode behavior and is always on.
- **Persistent state**: survives context destruction or reload; it requires a persistence backend
  (for example `localStorage`) and is optional unless the app explicitly configures it.

Persist, keyed by stable widget/window ID: window position, size, collapsed state; tree-node
open/closed; active tab selection; table column widths and sort spec; scroll position. State keys
must remain stable across frames using the same ID rules as §6.6 (including the reorder caveat).
`localStorage` persistence across page reloads is a reasonable optional backing store — not
mandatory unless the app specifically needs it to survive a reload, since runtime persistence across
frames is what the core immediate-mode contract actually requires.

### 6.23 Security `[P0]`

All user-supplied text (labels, table cell content, tooltip text, drag-drop payload display)
renders via `textContent`/DOM text nodes — **never** `innerHTML` or any HTML-interpreting sink.
Drag-drop payload data and table cell values are treated as data, never as markup or executable
content, regardless of what the application puts into them.

### 6.24 Project Structure `[P0]`

```
imgui-clone/
├── index.html
├── css/
│   ├── tokens-authentic.css     tokens-stylized.css     tokens-magenta.css (optional)
│   ├── components.css           layout.css
├── js/
│   ├── core/
│   │   ├── context.js           # §6.4 — the ImGuiContext-equivalent
│   │   ├── frame.js             # §6.5 — newFrame/render/lifecycle orchestration
│   │   ├── id-stack.js          # §6.6 — PushID/PopID, hashing, collision warnings
│   │   ├── layout.js            # §6.7 — cursor, SameLine, groups, item-width stack
│   │   ├── style-stack.js       # §6.13 — Push/PopStyleColor & Var
│   │   ├── input.js             # §6.9 — unified pointer/keyboard/wheel manager
│   │   ├── hit-test.js          # §6.10 — topmost-hoverable resolution, z-order, popup blocking
│   │   ├── persistence.js       # §6.22
│   │   └── dom-pool.js          # §6.20 — pooled node reconciliation
│   ├── widgets/
│   │   ├── button.js, drag.js, slider.js, checkbox.js, radio.js, combo.js,
│   │   │   color.js, progress.js, selectable.js, tree.js, table.js, tabs.js,
│   │   │   window.js, child-window.js, popup.js, tooltip.js, menu.js,
│   │   │   scrollbar.js, drag-drop.js, plot.js, splitter.js
│   ├── theme.js
│   └── utils.js
├── demo/
│   └── demo-window.js           # §6.26 — kitchen-sink reference, mirrors imgui_demo.cpp
└── tests/
    ├── unit/  integration/  visual/  performance/
```

### 6.25 JavaScript API `[P0]`

**Primary API — faithful immediate-mode**, corrected to fix the position-persistence bug and to
add the last-item/group/width primitives:

```javascript
function drawUI(app) {
  // cond defaults to "Once" — position seeds on first appearance only, then is
  // owned by persisted window state (drag/resize update it thereafter).
  if (!im.beginWindow('Settings', { x: 20, y: 20, w: 300 })) { im.endWindow(); return; }

    if (im.button('Save')) app.save();
    if (im.isItemHovered()) im.setTooltip('Writes settings to disk');

    im.pushItemWidth(-70);              // right edge sits 70px from window edge
    app.volume = im.slider('Volume', 0, 100, app.volume);
    app.scale  = im.drag('Scale', app.scale, { speed: 0.01, min: 0.1, max: 4 });
    im.popItemWidth();

    app.muted  = im.checkbox('Muted', app.muted);

    if (im.beginTable('Items', 3)) {
      const sortSpecs = im.tableGetSortSpecs();     // app-owned sort — table never reorders data
      if (sortSpecs?.dirty) app.items = sortBySpec(app.items, sortSpecs);

      app.items.forEach(item => {
        im.pushID(item.id);            // stable identity, NOT array index (see §6.6 reorder bug)
        im.tableNextRow();
        im.tableSetColumnIndex(0); im.text(item.name);
        im.tableSetColumnIndex(1); im.text(String(item.qty));
        im.tableSetColumnIndex(2); im.text(String(item.price));
        im.popID();
      });
      im.endTable();
    }
  im.endWindow();
}

function loop(timestamp) {
  im.newFrame(timestamp);
  drawUI(appState);
  im.render();
  requestAnimationFrame(loop);
}
requestAnimationFrame(loop);
```

**Retained convenience wrapper** (v2's builder shape, kept as a documented adapter — *not* a second
engine): internally, the wrapper holds its own `requestAnimationFrame` loop that calls the
immediate-mode `drawUI`-equivalent every frame on the wrapper's behalf, and fires the
builder-style callbacks by diffing each widget's boolean/changed return value frame-to-frame. This
is the concrete mechanism v2 left unspecified when it said the wrapper was "a thin adapter... not a
second parallel engine."

```javascript
const panel = ImGui.createWindow('My Panel', { width: 300, height: 200, x: 100, y: 100 });
panel.addButton('Click Me', () => console.log('Clicked!'));
panel.addCheckbox('Enable Feature', false, checked => console.log(checked));
```

### 6.26 Demo Application `[P0]`

A kitchen-sink demo covering every widget in §6.14 at least once, sectioned as: Basic widgets,
Inputs, Sliders/Drags, Colors, Trees, Tables (including a 10,000-row virtualized example), Tabs,
Menus, Popups/Modals, Tooltips, Drag/drop, Windows/Child windows, Style Editor, Accessibility
(tab-order walkthrough), Performance (a live widget-count stress toggle: 100/500/1000). This is the
project's living visual-regression reference, manual QA surface, and API example simultaneously —
build it incrementally alongside each phase (§7), not only at the end.

### 6.27 Documentation `[P0]`

`README.md` (including the non-goals from §6.0), API reference, architecture doc (context/frame
lifecycle/ID system), theme doc, widget doc, testing doc, a **known-deviations** doc (§10),
getting-started example, minimal-app example, advanced immediate-mode example.

### 6.28 Implementation Rules for a Coding Agent `[P0]`

1. Inspect the existing repository state before modifying anything; understand what's already built
   before adding to it.
2. Make changes incrementally, per phase (§7) — do not jump ahead to P1/P2 widgets before the P0
   context/input/layout/hover-active-focus foundation (§6.4–§6.10) is real and tested.
3. Run the relevant test tier (§8) after every meaningful change; check the browser console for
   warnings (§6.21) as well as errors.
4. Never leave a `TODO` in place of required P0/P1 behavior, and never hardcode demo-only behavior
   into a core widget module.
5. A feature is not "done" on visual appearance alone — see the No-Fake-Completion rule below and
   the Definition of Done (§9).
6. Update the relevant doc (§6.27) in the same change that alters a public API.
7. Never silently remove existing functionality; if a browser limitation blocks exact fidelity,
   document it (§10) rather than quietly shipping a different behavior.
8. Follow: **Understand → Plan → Implement → Test → Verify → Document** for each feature.

**No-fake-completion, made concrete** (these are not implemented until the *full* bullet is true):
- A slider that visually moves but doesn't update the bound value, or doesn't support Ctrl+Click
  entry, is not implemented.
- A popup that appears but doesn't handle focus-trapping (modal), click-outside, Escape, or the
  nested-stack closing rule (§6.14.15) is not implemented.
- A table that looks like a table but can't resize or sort columns, or that sorts the caller's data
  itself instead of exposing sort-specs, is not implemented.
- A drag widget that only responds to click, not continuous scrub-while-held, is not implemented.
- A theme selector that requires touching JS for a new preset is not a tokenized theme system.

### 6.29 Known Deviations — see §10 (kept as one canonical list, not duplicated here).

### 6.30 Acceptance Checklist — see §9 (kept as one canonical list, not duplicated here).

---

## 7. Implementation Roadmap

Dependency-ordered; do not start a phase before its prerequisites are real and tested.

| Phase | Builds | Depends on | Exit criteria |
|---|---|---|---|
| **0 — Architecture + Tokens** | Repo/project structure (§6.24), token CSS files (§6.2/6.3) | — | Presets swap with zero JS changes. Basic token infrastructure is established before runtime widgets are built. |
| **1 — Context + Frame Lifecycle** | `context.js`, `frame.js` (§6.4/6.5) | 0 | `newFrame()`/`render()` loop runs; frame count/time advance; no widgets yet. |
| **2 — Input + Hit Testing** | `input.js`, `hit-test.js` (§6.9/6.10) | 1 | A single invisible test rect correctly reports Hovered/Active/click through a full drag-out-to-cancel sequence. |
| **3 — Layout** | `layout.js`: cursor, SameLine, groups, item-width (§6.7) | 2 | Two dummy items lay out correctly with SameLine, group, and PushItemWidth combinations. |
| **4 — ID System** | `id-stack.js` (§6.6) | 3 | Duplicate-ID warning fires in dev build; `##`/`###` semantics pass their tests; stable-key ID generation is verified. The full reorder-state regression test is completed in Phase 5 when stateful widgets exist. |
| **5 — Core P0 Widgets** | Button, Checkbox, Radio, Text, Slider, Drag, TextInput | 2–4 | Each satisfies the Widget Contract (§6.12) end to end, including keyboard, and the full reorder-state regression test passes. |
| **6 — Windows** | `window.js`, `dom-pool.js`, persistence (§6.8/6.20/6.22) | 1–5 | Window drags, resizes, collapses, and **persists position across the exact `cond`-seeded example in §6.25** without regressing on every subsequent frame. |
| **7 — Popups/Modals/Tooltips** | `popup.js`, `tooltip.js` (§6.14.15) | 6 | Nested-stack closing rule test passes; modal blocks input behind it; tooltip timing configurable. |
| **8 — Tables** | `table.js`, virtualization (§6.14.12/6.20) | 6–7 | Sort-spec flows to the app and back without the widget mutating data itself; 10,000-row demo scrolls at 60fps. |
| **9 — Menus/Tabs** | `menu.js`, `tabs.js` (§6.14.13/6.14.14) | 6–7 | Submenu diagonal safe-zone verified; tab overflow scroll arrows work. |
| **10 — Theme System + Style Stacks** | Style/color/width stacks (§6.13) | 0–9 | Theme swap test (§8) passes on the full demo. Required: CSS token system + runtime style stacks. Optional: Style Editor (P3). |
| **11 — Accessibility Hardening** | ARIA layer, focus rings, tab-order reassignment (§6.15/6.17) | 5–9 | Full demo is Tab-navigable; high-contrast preset available. |
| **12 — Performance** | Node pooling audit, cleanup pass (§6.20) | 5–11 | 500-widget and 10,000-row benchmarks meet target; no detached listeners after 100 open/close cycles. |
| **13 — Testing** | Full suite (§8) | all prior | All tiers green. |
| **14 — Demo + Docs** | `demo-window.js` (§6.26), documentation set (§6.27) | all prior | Demo exercises every §6.14 widget at least once. |
| **15 — Fidelity Polish** | DPI verification (§6.18), remaining P2/P3 items | all prior | DPI matrix (§6.18) passes at 100/125/150/200%. |

---

## 8. Testing Strategy

**Unit** — ID hashing/collision detection, `##`/`###` semantics, style/width stack balance, layout
cursor math (SameLine/group/indent), hit-testing against nested clip rects, drag/slider value math
(including Ctrl+Click conversion and clamping), popup nested-stack open/close logic, table sort-spec
generation, table column-width distribution (Fixed/Stretch/AutoResize).

**Integration** — button drag-out-to-cancel (pointerdown → move off rect → pointerup fires no
click); full drag/slider pointer-capture sequence past widget, window, and viewport bounds (§6.11);
text input (focus, selection, paste, Escape-cancels-drag-edit); popup/modal open-close-Escape-
click-outside matrix, including the nested-stack case; menu submenu hover-open with diagonal safe
zone; tab switch and reorder; window drag/resize/collapse *and* the `cond`-seeded-position-then-
user-drag sequence from §6.25; **the ID reorder bug test**: build a list, focus/type into row 3's
text field, delete row 1, assert the typed text is still attached to the same underlying item, not
to whatever now displays as row 3.

**Visual** — screenshot comparison at fixed viewport size, browser, zoom, font, and
`devicePixelRatio`, across both theme presets and the `authentic`/`comfort` motion settings.

**Performance** — 100/500/1000 live widgets; a 10,000-row virtualized table scrolling continuously;
100 open/close cycles of a menu, popup, and modal each, checked for detached listeners/retained
closures afterward; repeated drag operations checked for accumulated floating-point drift.

**Regression** — every bug found during implementation gets a permanent, named test in the suite
before the fix is considered done — this specifically includes the window-position-`cond` bug and
the ID-reorder bug identified in this review, since both are exactly the class of bug that reappears
silently if untested.

---

## 9. Final Definition of Done

A feature is complete only when **all** of the following are true — visual completeness alone does
not qualify (§6.28's "No fake completion" rule):

- [ ] Public API exists and matches the documented shape (§6.25).
- [ ] Behavior is correct per its spec section, including edge cases named there.
- [ ] Visual appearance matches the active theme's tokens (§6.2/6.3) exactly, not eyeballed.
- [ ] Keyboard behavior works where the widget has any (§6.15).
- [ ] Accessibility requirements are met where applicable (§6.17).
- [ ] Relevant state persists correctly across frames/reloads where specified (§6.22).
- [ ] No leaked listeners/closures after repeated open-close or mount-unmount cycles (§6.20).
- [ ] Unit + integration tests exist and pass (§8).
- [ ] The widget appears in the demo application (§6.26).
- [ ] Documentation is updated (§6.27).

**Project-level gate**: every checklist item in this section is true for every P0 requirement, and
for every P1 requirement the team has committed to shipping, before calling the project "done" —
P2/P3 items may ship partially or not at all without blocking that gate, provided §6.0's non-goals
and any skipped P2/P3 items are recorded in the known-deviations doc.

---

## 10. Known Browser Limitations

Documented up front so "faithful clone" is understood as "as close as a browser allows," not
literally byte-identical to the C++ library:

- **Font rendering**: native ImGui bakes its font into a texture atlas and draws textured
  triangles at exact subpixel offsets via `stb_truetype`; a browser rasterizes text through its own
  engine/hinting. Expect sub-pixel differences even with an identical font file.
- **The pixel/bitmap font look is an approximation, not a guarantee** (corrected from v2, §6.3):
  `image-rendering: pixelated` only affects raster/canvas image scaling, not vector glyph
  rendering, and `font-smooth: none` is non-standard and ignored by current Chromium. A genuinely
  pixel-accurate bitmap look requires an actual pre-rasterized bitmap font asset; a vector
  "pixel-style" webfont is a labeled approximation, not the real thing.
- **Text editing internals**: native `<input>`/`<textarea>` IME composition, OS-level text
  selection, and caret rendering follow the browser's own implementation rather than ImGui's
  hand-rolled STB-textedit — this is a case where the browser's behavior is *better* (real IME/OS
  accessibility support) than native ImGui's, not a flaw to work around.
- **No true multi-viewport**: separate OS-level windows require an Electron/Tauri-style shell; out
  of scope for a plain browser tab (§6.0).
- **No docking branch**: real ImGui's drag-to-dock panel system is a separate branch from core
  Dear ImGui and is out of scope; Splitters (§6.14.19) are the supported alternative for fixed
  multi-pane layouts (§6.0).
- **No true GPU immediate-mode batching**: the DOM has a fundamentally different performance model
  than ImGui's vertex-buffer draw lists; the pooled-DOM approach (§6.20) approximates the *API*
  behavior, not the renderer internals.
- **Drag-and-drop ghost/payload rendering** uses native browser drag events or a manually
  positioned floating element, which will look and feel slightly different from ImGui's own
  draw-list-rendered tooltip-style payload preview.
- **Pointer capture cannot follow the pointer past the OS browser window's own edge** — capture
  works correctly to the edges of the browser viewport (§6.11) but not beyond the application
  window itself, unlike a native GPU app that owns the whole screen's input.
- **No literal zero-latency frame presentation**: any DOM/CSS pipeline has an unavoidable
  browser-composited-frame display latency that ImGui's direct-to-backbuffer draw doesn't have;
  negligible in practice, but not literally identical.

### v4 — Implementation-Contract Complete (hardening pass before final handoff)

This document is the authoritative implementation specification. No implementation should begin until the agent has completed the contradiction/ambiguity audit and resolved every issue it finds.

The remaining risk is not missing features — it is Claude interpreting unspecified details differently.

The goal of v4 is not to add more widgets. The goal is to lock down the exact contracts, state shapes, lifecycle ordering, and ownership rules that an implementation agent must follow.

#### 1. Define every public API precisely

Every public API must have an exact contract table. This is the biggest remaining gap.

| API | Parameters | Returns | State changes | Throws? | Notes |
|---|---|---|---|---|---|
| `beginWindow()` | `name, options` | `boolean` | window state | Never in production | Must pair with `endWindow()` |
| `button()` | `label, options?` | `boolean` | active/focus state | Never | `true` only on activation |
| `slider()` | `label, min, max, value, options?` | `number` | widget value | Never | Clamped |
| `pushID()` | `string|number|object` | `void` | ID stack | Dev warning | Must pair with `popID()` |
| `popID()` | none | `void` | ID stack | Dev warning | Must match `pushID()` |
| `beginChild()` | `id, size, options?` | `boolean` | child-window state | Never | Must pair with `endChild()` |
| `openPopup()` | `id, options?` | `void` | popup stack | Never | Opens popup at current stack depth |
| `beginPopup()` | `id, options?` | `boolean` | popup state | Never | Used for popup body; must match `endPopup()` |
| `destroy()` | none | `void` | clears listeners/state/maps | Never | Full teardown |

This must be enforced for every widget function, not only the examples. The implementation must not invent undocumented APIs, silently rename documented ones, or change parameter/return semantics.

The final public symbol list must be derived from the specification before implementation begins. At minimum, the specification shall include the following public surface area, with exact signatures and semantics:

```text
createContext
init
destroy
newFrame
render

beginWindow
endWindow
beginChild
endChild

pushID
popID

button
smallButton
arrowButton
checkbox
radioButton
selectable
text
inputText
inputTextMultiline
slider
drag

beginTable
endTable
tableNextRow
tableSetColumnIndex
tableGetSortSpecs

beginPopup
endPopup
openPopup
closeCurrentPopup
beginTooltip
endTooltip

beginMenu
endMenu
menuItem

beginTabBar
endTabBar
beginTabItem
endTabItem

beginDisabled
endDisabled

beginGroup
endGroup

sameLine
newLine
spacing
dummy
indent
unindent

pushItemWidth
popItemWidth
calcItemWidth

pushStyleColor
popStyleColor
pushStyleVar
popStyleVar

isItemHovered
isItemActive
isItemFocused
isItemClicked
isItemEdited
getItemRect

isMouseDown
isMouseClicked
isMouseReleased
isMouseDoubleClicked

setTooltip
```

#### 2. Fix the lifecycle ambiguities explicitly

§6.5 is close, but it still blurs together what happens before, during, and after the frame. Make this a formal lifecycle section:

- Before frame
  - input snapshot
  - reset transient state
  - prepare render/pool
- During frame
  - layout
  - hit testing
  - widget interaction
  - DOM declaration
- After frame
  - finalize hover
  - finalize clicks
  - popup closing
  - persistence
  - DOM cleanup
  - render

This should be written as a strict ordering contract. "Hit-test finalization" and "Interaction post-processing" must be described as separate phases, with explicit timing relative to widget declaration and DOM finalization.

#### 3. Add an explicit DOM ownership model

The implementation needs a DOM tree contract, not just a performance note.

```text
Context
└── Root DOM container
    ├── Window layer
    │   ├── Window A
    │   │   ├── Title bar
    │   │   └── Content
    │   └── Window B
    ├── Popup layer
    ├── Modal layer
    ├── Tooltip layer
    └── Drag/drop layer
```

Then define:

- Which DOM nodes persist
- Which nodes are pooled
- Which nodes belong to a widget
- When nodes are returned to the pool
- How a widget finds its existing node
- What happens when a widget disappears
- Whether DOM order equals declaration order
- How z-index is assigned
- Whether hidden nodes remain mounted

The implementation must not build a completely retained DOM framework by accident. Under a pooled immediate-mode model, the DOM tree is a frame-level projection, not a permanent object graph.

#### 4. Explicitly define z-index

Z-order should be deterministic and part of the spec, not left as an implementation detail. Add a formal stacking order:

```text
1. Base windows
2. Focused window
3. Popup layer
4. Submenu layer
5. Tooltip layer
6. Modal overlay
7. Modal content
8. Drag/drop preview
```

**Modal rule**: a modal is a global blocking layer. While a modal is open, hit-testing is restricted to the modal layer and its contents. Tooltip and popup interaction behind the modal is disabled until the modal closes.

Also define:

- whether a modal is above tooltips
- whether a tooltip can ever appear over a popup
- whether a popup can appear above a modal
- how click-through and hit-testing work at each layer

If these are not specified, Claude Code will guess and build inconsistent stacking behavior.

#### 5. Add a formal widget state-transition table

The current state machine is conceptual but should be explicit. Add exact transitions.

```text
Idle
├─ pointerdown + hovered → Active
├─ keyboard focus → Focused
└─ disabled → Disabled

Active
├─ pointermove outside → Active
├─ pointerup + hovered → Activated → Idle
├─ pointerup outside → Idle
├─ Escape → Cancelled → Idle
└─ pointercancel → Idle

Focused
├─ Tab → next focusable
├─ Shift+Tab → previous focusable
├─ Enter/Space → Activated
└─ Escape → widget-specific cancel
```

Also define explicitly what happens when focus changes while `activeId` exists, and which side wins: the focus-change, the active drag, or the pointer capture.

#### 6. Add a formal API compatibility matrix

Add a compatibility table like this, and expand it beyond the example.

| ImGui concept | Clone API | Fidelity |
|---|---|---|
| `Begin()` | `beginWindow()` | High P0 |
| `End()` | `endWindow()` | High P0 |
| `BeginChild()` | `beginChild()` | High P0 |
| `PushID()` | `pushID()` | High P0 |
| `PopID()` | `popID()` | High P0 |
| `Button()` | `button()` | High P0 |
| `Checkbox()` | `checkbox()` | High P0 |
| `SliderFloat()` | `slider()` | High P0 |
| `DragFloat()` | `drag()` | High P0 |
| `InputText()` | `inputText()` | High P0 |
| `BeginTable()` | `beginTable()` | High P1 |
| `TableGetSortSpecs()` | `tableGetSortSpecs()` | High P1 |
| `BeginPopup()` | `beginPopup()` | High P0 |
| `OpenPopup()` | `openPopup()` | High P0 |
| `BeginTooltip()` | `beginTooltip()` | High P0 |
| `BeginMenu()` | `beginMenu()` | High P1 |
| `BeginTabBar()` | `beginTabBar()` | High P1 |
| `Columns()` | none | Intentional non-goal |
| Docking | none | Intentional non-goal |
| Multi-viewport | none | Browser limitation |

This makes it explicit which APIs are intentionally missing and which are intentionally equivalent.

#### 7. Add browser automation requirements

Do not describe testing abstractly. Require actual Playwright-based browser automation:

```text
tests/
├── unit/
├── integration/
├── visual/
├── performance/
├── e2e/
│   ├── windows.spec.js
│   ├── widgets.spec.js
│   ├── keyboard.spec.js
│   ├── popups.spec.js
│   ├── tables.spec.js
│   └── persistence.spec.js
└── fixtures/
```

Required automated tests:

- Window dragging
- Window resizing
- Slider dragging
- Drag-out cancellation
- Ctrl+Click numeric editing
- Tab navigation
- Shift+Tab
- Escape
- Popup nesting
- Modal blocking
- Tooltip delay
- Table sorting
- Table virtualization
- Stable IDs after deletion
- Theme switching
- Persistence after reload
- 10,000-row scrolling
- 500/1000-widget stress test

This should be explicitly listed under the testing strategy as required before a feature ships.

#### 8. Add initialization and destruction

The document should define the context lifecycle, not just the frame lifecycle:

```javascript
const im = ImGui.createContext({
  root: document.querySelector('#app'),
  theme: 'authentic',
});

im.init();

// ... drawUI...

im.destroy();
```

`destroy()` must explicitly:

- remove global event listeners
- release pointer capture
- clear timers
- remove pooled DOM nodes
- clear state maps
- cancel pending animation frames
- remove resize observers
- release references that could prevent garbage collection

This pairs directly with the memory-leak requirements already in the document.

#### 9. Add a "Do Not Invent API" rule

Add an explicit implementation rule:

> ### No Invented Public APIs [P0]
>
> The implementation MUST NOT invent undocumented public APIs, silently rename documented APIs, or change documented parameter or return-value semantics.
>
> If an implementation requires a new public API that is not specified here, the agent MUST stop before exposing it publicly and document the proposed API addition.
>
> Private/internal helper functions may be created freely.

This is extremely valuable for a project of this size. It prevents the common failure mode where a code agent quietly adds a convenience API that later becomes a compatibility trap.

#### 10. Add a contradiction audit as the final step

Before Claude Code receives the spec, require one final contradiction audit:

> Search the entire specification for contradictions, ambiguous requirements, duplicated responsibilities, impossible browser requirements, undefined terms, missing dependencies, APIs referenced but never defined, requirements that conflict with the roadmap, and acceptance criteria that cannot actually be measured. Fix every issue found. Do not merely list the problems.

This should be the last step before implementation begins.

#### 11. Define the exact data model for important objects

You have excellent descriptions of `WindowState`, `WidgetState`, `lastItemData`, `popupStack`, and other core structures, but Claude can still invent their internal shapes. Require that every persistent or core state structure define:

- exact fields
- field types
- required vs. optional fields
- default values
- who creates it
- who mutates it
- when it is reset
- whether it survives frames
- whether it survives reloads
- whether it is public or private

For example:

```ts
WindowState {
  id: string
  x: number
  y: number
  width: number
  height: number
  minWidth: number
  minHeight: number
  collapsed: boolean
  open: boolean
  scrollX: number
  scrollY: number
  zIndex: number
  focused: boolean
  appearing: boolean
  dirty: boolean
  persistenceEnabled: boolean
}
```

Do the same for `WidgetState`, `PopupState`, `LastItemData`, `IOState`, `DragDropState`, `TableState`, and the various stacks. This is what prevents Claude Code from filling in architectural gaps on its own. It also ensures the implementation does not silently invent a different shape for what the spec calls "window state" or "popup state."

#### 12. Define actual API signatures and option enums

The implementation must define a complete signature table for every public function, not just a handful of examples. Each public symbol must specify:

- parameters
- options shape
- return type
- mutation behavior
- state changes
- activation conditions
- errors/recovery
- example usage

Example format:

```text
button(label, options?)
  Returns: boolean
  Activates: pointer click or keyboard Enter/Space
  Mutates: active/focus state only
  Options: size?, disabled?, tooltip?

slider(label, min, max, value, options?)
  Returns: number
  Mutates: temporary interaction state + returned value
  Options: speed?, format?, width?, disabled?

inputText(label, value, options?)
  Returns: string
  Mutates: returned value + text-edit state
  Options: width?, placeholder?, multiline?, flags?

beginWindow(name, options?)
  Returns: boolean
  Options: x?, y?, w?, h?, cond?, flags?
```

Enum names and option names must be defined in a single canonical namespace. The project must pick one format and use it consistently:

```text
ImGuiWindowFlags.NoResize
ImGuiWindowFlags.NoMove
ImGuiWindowFlags.NoTitleBar

ImGuiCond.Once
ImGuiCond.FirstUseEver
ImGuiCond.Appearing
ImGuiCond.Always
```

The same applies to table sizing, input flags, popup flags, and colors. Claude Code may not invent alternative option names or ad hoc flag objects that silently change semantics.

#### 13. Define coordinate systems and DPI behavior

All geometry must use an explicit coordinate model:

- Screen coordinates: origin = viewport top-left; units = CSS pixels.
- Window coordinates: origin = window content top-left; units = CSS pixels.
- Item rectangles: stored in screen coordinates after layout.
- Mouse coordinates: `io.mousePos` uses screen CSS pixels.
- Canvas coordinates: logical CSS pixels for drawing; device pixels only for backing buffers.

This removes ambiguity around scaling, drag math, and DPI bugs. For example, a window must not move twice as far at 200% scaling because one subsystem treats values as CSS pixels and another treats them as device pixels.

#### 14. Define clipping behavior

Every window and child window owns a clip rectangle.

A widget is:

- layout-visible if its rect intersects the current clip rect
- hoverable only if its rect intersects the clip rect
- rendered only if it intersects the clip rect

Child clip rectangles intersect their parent's clip rectangle.

A clipped widget must not claim `hoveredId` or `activeId` solely because its geometric rectangle contains the mouse.

This requirement is especially important for scrolling, virtualization, and tables.

#### 15. Define scroll behavior precisely

The implementation must define all scroll rules explicitly:

- `scrollX` / `scrollY` units are CSS pixels
- minimum scroll is `0`
- max scroll is `max(0, contentSize - viewportSize)`
- wheel sensitivity is explicit and constant per input configuration
- scrollbar dragging uses the same clamped coordinate math as programmatic scroll
- if content becomes smaller than the current scroll position, scroll clamps to the valid range
- scroll position may persist across frames and reloads if the relevant state is being persisted

The canonical formula is:

```text
scrollX = clamp(scrollX, 0, max(0, contentWidth - viewportWidth))
scrollY = clamp(scrollY, 0, max(0, contentHeight - viewportHeight))
```

#### 16. Define the actual focus model

The implementation must distinguish the following values and treat them as related-but-not-equal:

- `focusedWindowId` = window receiving keyboard navigation
- `navId` = currently keyboard-navigable ImGui item in that window
- `document.activeElement` = browser-native DOM focus
- `activeId` = currently captured interaction

These values are related but must not be treated as aliases.

#### 17. Define pointer IDs and multi-pointer behavior

Because the implementation uses Pointer Events, it must define deterministic multi-pointer rules:

- only one pointer may own `activeId` at a time
- the pointer that claims `activeId` becomes `activePointerId`
- other pointer events cannot modify the active interaction
- `pointerup` / `pointercancel` only release `activeId` when `pointerId === activePointerId`

This makes pointer capture deterministic and prevents a second pointer from breaking a drag in progress.

#### 18. Define browser-event → ImGui-input conversion

The input manager must explicitly handle at least the following events:

- `pointerdown`
- `pointermove`
- `pointerup`
- `pointercancel`
- `wheel`
- `keydown`
- `keyup`
- `input`
- `compositionstart`
- `compositionupdate`
- `compositionend`
- `paste`
- `contextmenu`
- `blur`
- `visibilitychange`
- `resize`

The implementation must define what happens when the browser loses focus or visibility while a drag is active. In particular, blur/visibilitychange must release or cancel the relevant interaction deterministically.

#### 19. Define invalid call and recovery behavior

The implementation must define the behavior of invalid calls explicitly.

Example rules:

```text
Development:
  console.warn with API name, frame number, and current scope.

Production:
  recover safely without throwing.

Examples:
  endWindow() with empty windowStack -> no-op
  popID() with empty ID stack -> no-op
  popStyleColor() with empty stack -> no-op
```

A widget stack must never silently corrupt the global state because one stack was mismatched.

#### 20. Define what "same frame" means

> All widget return values, hover states, active states, edited states, and item rectangles describe the widget's state during the current `newFrame()` → `render()` interval. No public widget interaction result may depend on DOM state produced by a previous frame.

This rule prevents the implementation from accidentally becoming retained-mode by relying on a previously rendered DOM node's state instead of the current frame's immediate-mode calculations.

#### 21. Add resize observer requirements

The implementation may use `ResizeObserver` for:

- root viewport size
- externally resized host containers

It must not use `ResizeObserver` for:

- measuring every widget
- determining widget layout
- replacing the layout engine

This keeps layout and hit-testing inside the immediate-mode engine rather than letting a DOM observer become a second source of truth.

#### 22. Define persistence versioning and storage namespacing

Persistence must be versioned and namespaced:

```json
{
  "version": 1,
  "windows": {},
  "widgets": {},
  "tables": {}
}
```

Rules:

- unsupported versions are ignored or migrated explicitly
- migration must be versioned and deterministic
- corrupted persisted data must not prevent the UI from starting
- storage keys must be namespaced to the application (`<application-id>:<imgui-context-id>:state`), not a generic key such as `imgui-state`

This prevents different applications on the same origin from overwriting each other's persisted UI state.

#### 23. Define deterministic testing environment requirements

Visual and browser tests must lock the environment to deterministic values:

- browser: Chromium
- viewport: fixed dimensions
- zoom: 100%
- devicePixelRatio: explicit value
- font availability: verified
- animations: disabled
- locale: fixed
- timezone: fixed
- reduced-motion preference: fixed

Without this, screenshot tests can drift across machines for reasons unrelated to the implementation.

#### 24. Add the single-source-of-truth rule

> No duplicated state authority.
>
> For every piece of state, exactly one subsystem owns the authoritative value.
>
> Examples:
> - Window position → `WindowState`
> - Widget interaction → context interaction state
> - Input → `InputState` / `IO`
> - Theme → CSS token system
> - Table ordering → application-owned data model
> - Persistent settings → persistence subsystem
> - DOM nodes → DOM pool/reconciler
>
> DOM state must never silently become authoritative over library state.

This is the main safeguard against Claude creating a messy blend of retained DOM state, widget object state, persistence state, and context state all competing to define the same fact.

### Handoff instructions to Claude Code

Before work begins, the implementation agent MUST do the following:

1. Read this entire specification before modifying the repository.
2. Treat this document as the authoritative implementation contract.
3. Inspect the existing repository and compare it against this specification.
4. Identify what already exists and what is missing.
5. Produce a dependency-ordered implementation plan before making any code changes.
6. Do not invent public APIs.
7. Do not remove existing functionality.
8. Do not begin P1/P2/P3 work before the P0 foundation exists.
9. Use the phase order defined in §7 and report progress after each phase.

Implementation order:

Phase 0 → Architecture
Phase 1 → Context + Frame Lifecycle
Phase 2 → Input + Hit Testing
Phase 3 → Layout
Phase 4 → IDs
Phase 5 → Core Widgets
Phase 6 → Windows
Phase 7 → Popups
Phase 8 → Tables
Phase 9 → Menus/Tabs
Phase 10 → Themes
Phase 11 → Accessibility
Phase 12 → Performance
Phase 13 → Testing
Phase 14 → Demo/Docs
Phase 15 → Fidelity Polish

After each phase, the agent must execute: Implement → Test → Verify → Report, then proceed only if the phase passes.

### Final verdict

This document is the authoritative implementation specification. No implementation should begin until the agent has completed the contradiction/ambiguity audit and resolved every issue it finds.

At this point, the architecture is good enough that it should not be expanded with another large swarm of feature ideas. The spec is already strong on the core architecture:

Context → Frame → Input → IDs → Layout → State → Widgets → Windows → Popups → DOM → Persistence → Testing

The remaining work is refinement: define every API contract exactly, lock down lifecycle ordering, define DOM ownership and z-order, add deterministic state machines, and require actual browser automation before shipping.

This is the difference between "a strong implementation brief" and "a spec that is safe enough for automated coding."


