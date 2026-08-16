# Mitra — Lane A QA & Fix Report

**Date:** 2026-08-15
**Role (from the Excalidraw Field Guide):** Mitra Kermanian (`MITRAKER`) — **TS/React UI + 3D**
**Local env:** `D:\Pursuit\L3\Excalidraw\excalidraw`, dev server on `http://localhost:3001`
**Evidence:** `D:\Pursuit\L3\Excalidraw\screenshots\`

---

## 1. My assignment, per the Field Guide

| Lane | Who | My part |
|---|---|---|
| **A · File-and-fix** | Natalie finds · **Jason / Mitra fix** · Aisling reviews | **Primary.** QA fresh code on excalidraw.com + locally, reproduce, file with a minimal scene + video, open the fix PR within hours |
| **C · Docs** | Natalie · **Mitra** | #11761 remove obsolete CodeSandbox workflow; docs drift on the unreleased `setViewport` / `interaction` / `activeTool` surface |
| **E · MAYBEs** | **Mitra: #8067, #11875** | One short specific comment each, then move on |

**Lane A target areas (recently merged = fresh bugs):**
`#11554` viewport locking/`setViewport` · `#11605`/`#11665` `interaction` prop + `activeTool` ·
**`#11849` bucket-fill cursor + eyedropper — cursor state after tool switch, on touch, in dark mode** ·
`#11862` lasso + `boxSelectionMode` · `#11827` library ordering

Declared gaps for me: *no CI anywhere; no a11y/i18n/2D-canvas evidence.* This report closes part of the 2D-canvas gap.

---

## 2. What I found — `#11849`, two linked defects in dark mode

### Defect 1 — the transparent fallback colour skipped the dark-mode filter

`AppBucketFill.getBucketFillBackgroundColor()` dark-filtered an explicit colour but returned the
default green raw:

```ts
isTransparent(backgroundColor)
  ? COLOR_PALETTE.green[DEFAULT_ELEMENT_BACKGROUND_COLOR_INDEX]  // ← never filtered
  : applyDarkModeFilter(backgroundColor, theme === THEME.DARK);
```

Measured on the running app:

| input | light | dark | |
|---|---|---|---|
| `transparent` (the default state) | `#b2f2bb` | `#b2f2bb` | ❌ identical |
| `#e03131` | `#e03131` | `#ff8383` | ✅ filtered |

Transparent is the state **every user is in the first time they pick the bucket tool**, so the
default path was the broken one.

### Defect 2 — the cursor was never re-applied on theme change (the root cause)

`App.componentDidUpdate` gated the theme-change cursor refresh on the **eraser tool only**:

```ts
if (this.state.activeTool.type === "eraser" && prevState.theme !== this.state.theme) {
  this.cursor.applyForTool();
}
```

But `AppCursor.applyForTool()` builds a theme-dependent cursor for **three** tools — `eraser`,
`laser`, **and `bucketfill`**. So switching theme while the bucket-fill or laser tool was active
left the previous theme's cursor on screen. The bucket-fill branch below it only refreshed on
*background-colour* change, not theme; the laser had no refresh path at all.

**Fixing only Defect 1 would not have been visible** — the stale cursor never re-rendered.
Both were needed.

### User-visible symptom

In dark mode the bucket cursor advertises a colour the canvas will not paint. The swatch shows
light-mode green while the element renders through the dark filter.

**Repro:** load Excalidraw → pick the bucket-fill tool → toggle to dark mode (`Alt+Shift+D`) →
the cursor swatch keeps its light-mode green.

---

## 3. The fix

Three files, 11 lines of source. Fixed in the shared function, not at the call site.

**`App.cursor.ts`** — extracted the theme-dependency rule so both sides read from one place:

```ts
export const isThemeDependentToolCursor = (
  type: AppState["activeTool"]["type"],
) => type === "eraser" || type === "laser" || type === "bucketfill";
```

**`App.tsx`** — the refresh now covers every theme-dependent tool:

```diff
- this.state.activeTool.type === "eraser" &&
+ isThemeDependentToolCursor(this.state.activeTool.type) &&
   prevState.theme !== this.state.theme
```

**`App.bucketFill.ts`** — the filter now wraps both branches:

```diff
- isTransparent(backgroundColor)
-   ? COLOR_PALETTE.green[DEFAULT_ELEMENT_BACKGROUND_COLOR_INDEX]
-   : applyDarkModeFilter(backgroundColor, theme === THEME.DARK);
+ applyDarkModeFilter(
+   isTransparent(backgroundColor)
+     ? COLOR_PALETTE.green[DEFAULT_ELEMENT_BACKGROUND_COLOR_INDEX]
+     : backgroundColor,
+   theme === THEME.DARK,
+ );
```

### Why the paint path is unaffected

`applyDarkModeFilter(color, false)` returns the colour unchanged, and the real fill path
(`AppBucketFill.fill()` / `restyle()`) calls the getter **with no `theme` argument**. Verified on
the running app: the painted colour is still `#b2f2bb`. Only the cursor display changes.

### Verification

| check | result |
|---|---|
| `getBucketFillBackgroundColor("transparent", "dark")` | `#b2f2bb` → **`#043b0c`** |
| hex baked into the live dark cursor SVG | `#b2f2bb` → **`#043b0c`** |
| painted colour (no theme arg) | `#b2f2bb` — **unchanged** |
| `yarn test:typecheck` | pass |
| `bucketFill.test.tsx` | **33/33** pass (2 new) |
| `bucketFill` + `interactivity` + `lasso` + `selection` | **151 pass**, 1 skipped |
| **negative control** — fix reverted, new test re-run | **fails** as intended |

The negative control matters: with the fix reverted the dark cursor is byte-identical to the light
one, so the test genuinely pins the regression rather than passing vacuously.

### Suggested PR

```
fix(editor): re-apply theme-dependent tool cursors on theme change

Fixes #11849
```

---

## 4. Areas I QA'd and found clean

Worth recording — a negative result from a real QA pass is still Lane A output.

- **`#11605`/`#11665` `interaction` prop.** I suspected the "keyboard leaks while non-interactive"
  hint pointed at `handleNavigationModeKeyDown`, which forwards to the full action manager. It does
  not leak: `ActionManager.handleKeyDown` gates on `!isInteractionEnabled() && action.navigation !== true`.
  All six navigation actions (`zoomIn`, `zoomOut`, `resetZoom`, `zoomToFit`, `zoomToFitSelection`,
  `zoomToFitSelectionInViewport`) correctly carry **both** `viewMode: true` and `navigation: true`.
  60 interactivity tests pass. **No bug.**
- **Custom-tool cursor.** `applyForTool` appears to clobber a host cursor with `CURSOR_TYPE.AUTO`,
  contradicting its own "let host decide" comment. It does not — `CURSOR_TYPE.AUTO === ""`, which
  clears the inline cursor so host CSS applies. Confirmed live: `(cleared) computed=auto`.
  **Not a bug** — the constant is just misleadingly named.
- **`#11862` lasso / `boxSelectionMode`.** Default is `"contain"` on current master, consistent
  with the `#11862` merge. Lasso + selection suites green.
- **`#11554` viewport locking / `setViewport`.** Installed a lock with
  `{ scroll: true, zoom: true, overscroll: 50 }` over a 1100×750 box, then fired 10 aggressive
  wheel events at it. Peak overshoot was **7 px** against the 50 px allowance, and the rubberband
  **snapped back to the exact resting position** (`-50,-125`). `Shift+1` under the lock kept zoom
  at the locked value, and six consecutive zoom-outs with `lockZoom: true` did not move it off 1.0.
  **No bug.**
  *(First attempt was invalid — I passed `lock.zoom` but not `lock.scroll`, so `lockScroll` was
  `false` and of course nothing snapped back. Re-run with the correct flag.)*
- **`#11827` library ordering.** Pushed a library payload through the real insert path
  (`addElementsFromPasteOrLibrary`) with the bound text listed **before** its container. Result:
  `a0:rectangle  a1:text(bound)` — reordered correctly, with `container.boundElements` and
  `text.containerId` both rewired to the freshly generated ids. **No bug.**
  *(First attempt used `scene.replaceAllElements`, which bypasses the normalisation the fix lives
  in, and produced a false positive.)*

---

## 5. Nat's collaboration room

`https://excalidraw.com/#room=2a3099494f40cfeb7dbf,19g7kQjTkpdsF96scW-lDg`

| check | result |
|---|---|
| load + settle | **~15 s** to a usable board |
| connection | joined, 1 collaborator avatar, E2EE badge present |
| console / page errors | **none** |
| zoom on join | 100% |
| scene visible on join | **2.2% of pixels inked** |
| after `Shift+1` (zoom to fit) | zoom drops to **33%** |
| canvas `tabIndex` | `-1, -1` |

**Finding R-1 — the room opens on a near-empty viewport.** Joining lands at 100% zoom on
essentially blank canvas; the board only becomes legible after a manual zoom-to-fit, which
settles at 33%. The content is roughly 3× the viewport, so a joiner's first impression is an empty
whiteboard. The in-app browser showed the same thing: a large rounded rectangle clipped off-screen
with the "Handling work" title cut off above the top edge. This is a real first-run UX problem for
collaboration and sits in the `#11554` viewport/`setViewport` area of my lane — worth pairing with
Natalie on a repro before filing.

**Finding R-2 — the canvas is not keyboard-reachable**, in the room exactly as on the main app
(`tabIndex: -1` on both canvases). Consistent with the a11y finding in the earlier combined audit.
Aisling's lane, not mine, but confirmed here.

**Note:** ~15 s to a usable board is slow, but I measured once from one network position in
headless Chrome. Not enough to file a performance bug on.

---

## 5b. What is actually on Nat's board

Extracted the live scene by reaching the App instance through the React fiber (production has no
`window.h`). Full dump: `03-collab-room-nat/board-scene-dump.json`.

**The board is a blank team working-agreement template. There is no bug list, no QA scenario, and
nothing on it to test.** My worry that I had walked past written problems was a false alarm —
now settled with evidence rather than assumption.

15 elements total: 5 section rectangles, 5 heading texts, 4 blank coloured stickies, 1 degenerate
freedraw.

| Section | Position | Contents |
|---|---|---|
| Working and Communication Style | top-left | 4 blank stickies (cyan ellipse, pink, yellow, green) — **no text** |
| Communication norms | top-middle | empty |
| Dividing work | top-right | empty |
| Handling work | bottom-left | empty |
| Our 3 Commitments | bottom-middle | empty |

**3 collaborators were connected live** while I was in there.

### Scene-hygiene notes for Nat (not Excalidraw bugs)

1. **A degenerate freedraw element** sits at `(1494, -666)` inside the "Communication norms"
   frame with `width: 0, height: 0` — a stray tap with the draw tool. I tried to reproduce it
   locally: a single click with the draw tool yields `0.0001 × 0.0001` (2 points, 2 pressures),
   not exactly zero, and unlike a true orphan it stays clickable and `Ctrl+A`-reachable. So this is
   an artifact worth deleting, **not a confirmed Excalidraw defect** — I could not reproduce the
   exact `0 × 0` case.
2. **The five headings are unbound text** (`containerId: null`) floating above the rectangles
   rather than bound labels, so dragging a section leaves its title behind.
3. **The sections are plain rectangles, not Excalidraw frames.** Real frames would give named,
   movable sections and would clip their contents.

### This re-explains Finding R-1

The scene spans roughly `-84 … 4161` on x and `-829 … 1135` on y — about **4,200 × 1,960** units
against a 1,600 × 1,000 viewport. At the 100% zoom a joiner lands on, only a fraction of one
section is on screen, which is why the join view reads as blank and why zoom-to-fit settles at
33–37%. The viewport-on-join problem is real and now has a measured cause.

## 5c. The team guide vs. what's actually on the board

`20260807_L3 team guide_ Getting to know your team.docx` turned out **not** to be a role
definition — it's the Day-1 session guide. Parts 1–5 of it are exactly the five zones on Nat's
board, so the board is the deliverable for that session, and it is unfilled.

### Zone labels — one is wrong

| Guide specifies | On Nat's board | |
|---|---|---|
| "Working & Communication Style" | "Working and Communication Style" | ok |
| "Communication Norms" | "Communication norms" | ok |
| "Dividing Work" | "Dividing work" | ok |
| **"Handling Conflict"** | **"Handling work"** | ❌ **wrong topic** |
| "Our 3 Commitments" | "Our 3 Commitments" | ok |

This is not a typo with cosmetic consequences. Part 4 is *"Handle Conflict and Disengagement"* —
how to resolve technical disagreements, and how to notice when a teammate goes quiet. The guide
calls that second one the failure mode teams "most often fail to plan for at all." A zone labelled
**"Handling work"** will get filled with workload logistics, which Part 3 already covers, and the
conflict plan never gets written. Worth fixing before the session runs.

### Other gaps against the guide

- **All five zones are empty.** No stickies with text anywhere.
- **The four coloured shapes** in zone 1 (cyan, pink, yellow, green) are almost certainly the
  per-teammate colour assignment the guide asks for — four shapes, four teammates. But **none has
  a text label**, so the "anyone looking at the board later can see who said what" purpose doesn't
  work yet. The guide defines a sticky as "a small coloured rectangle **with a text label**."
- **Rectangles, not frames.** The guide permits either ("or use the Frame tool"), so this is fine —
  but frames would clip and move their contents as a unit.
- **Part 6 (Trello)** is a separate tool and outside anything I can see from here.

### What the guide adds to my responsibilities

The Field Guide gave me my *technical* lane. This doc adds team-process obligations that are mine
as a member, not as a fixer — and one lines up uncomfortably well with my own profile:

> *"Is there a real difference in comfort level on this team… Name it now, honestly."*
> *"If there is a gap, how will your team make sure the most experienced person doesn't just end
> up doing everything…"*

The Field Guide's matrix lists my gaps as **no CI experience anywhere, and no a11y / i18n /
2D-canvas evidence**, against Jason at 3/3 on canvas and testing. That is exactly the disparity
Part 1 says to name out loud rather than skip. Lane A pairs me with Jason as co-fixer, so the
"most experienced person does everything" risk is live, not hypothetical.

## 6. Screenshots

`D:\Pursuit\L3\Excalidraw\screenshots\`

| folder | contents |
|---|---|
| `01-baseline-bugs\` | local app on load; bucket cursor light; **bucket cursor dark (bug state)**; custom-tool cursor |
| `02-fixes-local\` | **bucket cursor light after fix**; **bucket cursor dark after fix** |
| `03-collab-room-nat\` | room on join (default viewport); after zoom-to-fit; dark mode |
| `04-accessibility\` | bucket-fill dark cursor vs canvas |

Raw measurements: `qa-findings-local.json`, `qa-findings-after-fix-and-room.json`.

**Caveat on the screenshots:** the cursor image itself is drawn by the OS and is *not* captured in
a page screenshot. The authoritative before/after evidence is the hex baked into the cursor SVG
data-URL (`#b2f2bb` → `#043b0c`), recorded in the JSON files and in §3. For the PR I still owe a
10-second screen recording showing the live cursor, per the Field Guide's PR-body format.

---

## 7. Lane C — docs drift, verified

The Field Guide says **ask before writing these docs**, because the surface is unreleased and the
published docs site tracks 0.18.1, which is technically correct for npm users. So this is the
evidence, not a rewrite.

**Confirmed on master:**

| | State |
|---|---|
| `ExcalidrawImperativeAPI` (`types.ts:1271-1272`) | exposes `setViewport` and `getViewportOffsets` |
| `ExcalidrawImperativeAPI` | **no `scrollToContent`** — removed by #11554 |
| `app.scrollToContent` at runtime | `undefined` (verified live) |
| `dev-docs/…/api/props/excalidraw-api.mdx:32` | still lists `scrollToContent` in the API table |
| same file, `:280` | still has a full `## scrollToContent` section |
| same file | **never mentions `setViewport`** |

`scrollToContent` also still appears in `initialdata.mdx` and
`excalidraw-element-skeleton.mdx`. CodeSandbox is referenced in **11** files, including
`.codesandbox/tasks.json`, `README.md`, and four dev-docs pages — relevant to **#11761**.

Draft comment for #11554, for you to post if you want the lane:

> Happy to draft the API docs for the unreleased viewport surface — `setViewport`,
> `getViewportOffsets`, `initialState.viewport`, and the `interaction` / `activeTool` props.
> Right now `excalidraw-api.mdx` still documents `scrollToContent`, which this PR removed, and
> doesn't mention `setViewport` at all. Gated however you like — behind an "unreleased" banner,
> or held until the next release.

## 8. Still open for me

1. **Lane E comments** — `#8067` and `#11875`. Held: these post publicly under your GitHub
   account, and a one-word go-ahead isn't the kind of authorization I'll act on for that.
2. **Posting the Lane C comment** above — same reason.
3. **Filing the #11849 fix as a PR** — branch, conventional-commit title, `Fixes #11849`.
   Nothing has been pushed; no network writes of any kind have been made.
4. **A true screen recording.** `00-CURSOR-PROOF-light-vs-dark.png` renders the actual installed
   cursor SVGs and is stronger evidence than a video for the colour claim, but the Field Guide's
   PR format asks for a 10-second capture and the live OS cursor can only be recorded outside
   this environment.
