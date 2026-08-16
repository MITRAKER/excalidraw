# Ready to post — needs your GitHub session

**I could not post any of this.** No `gh` CLI on this machine, the GitHub MCP server can't run its
OAuth flow in a non-interactive session, and Chrome is signed out of GitHub
(`/settings/profile` redirects to the login page). Entering credentials is off-limits for me, so
these are staged for you rather than sent.

Everything below is final text. The fix is already committed locally.

---

## 1. The PR — one push away

Branch `fix/theme-dependent-tool-cursors`, commit `21f9be6`, 4 files, +81/−11.
Local clone: `D:\Pursuit\L3\Excalidraw\excalidraw` (its `origin` is upstream, so you need a fork).

**Fork `excalidraw/excalidraw` on GitHub first**, then:

```bash
cd "D:/Pursuit/L3/Excalidraw/excalidraw"
git remote add fork https://github.com/MITRAKER/excalidraw.git
git push -u fork fix/theme-dependent-tool-cursors
```

Then open the PR against `excalidraw:master`.

**Title** (CI enforces conventional titles):

```
fix(editor): re-apply theme-dependent tool cursors on theme change
```

**Body:**

> Fixes #11849
>
> ## Summary
>
> The theme-change cursor refresh in `componentDidUpdate` was gated on `activeTool.type === "eraser"`,
> but `AppCursor.applyForTool()` builds a theme-dependent cursor for **eraser, laser and bucketfill**.
> Switching theme while the laser or bucket-fill tool was active left the previous theme's cursor installed.
>
> Separately, `getBucketFillBackgroundColor()` dark-filtered an explicit background colour but returned
> the transparent fallback raw — and transparent is the default, so the bucket cursor advertised
> `#b2f2bb` while the dark canvas painted something else. Fixing only one of these is invisible: without
> the refresh the corrected colour never re-renders, and without the filter the refresh recomputes the
> same value.
>
> Extracted `isThemeDependentToolCursor()` so `App` and `AppCursor` share one definition instead of two
> lists that can drift, and wrapped `applyDarkModeFilter` around both branches of the getter.
>
> **The painted colour is unchanged.** The fill path calls the getter with no `theme` argument, and
> `applyDarkModeFilter(colour, false)` is identity — verified on the running app: still `#b2f2bb`.
>
> ## Before / After
>
> Bucket-fill tool, transparent background, dark mode — the cursor SVG the editor actually installs:
>
> | | light | dark |
> |---|---|---|
> | before | `#b2f2bb` | `#b2f2bb` ← unchanged by the theme |
> | after | `#b2f2bb` | `#043b0c` |
>
> <!-- attach screenshots/02-fixes-local/00-CURSOR-PROOF.gif (6s loop, before/after) -->
> <!-- static version: 00-CURSOR-PROOF-light-vs-dark.png -->
>
> *(These render the cursor SVG the editor installs — a page screenshot cannot capture the live
> OS cursor. The right-hand swatch is the colour the dark canvas actually paints.)*
>
> ## Tests
>
> Two added to `packages/excalidraw/tests/bucketFill.test.tsx`:
>
> - `re-applies the cursor when the theme changes` — pins the refresh. Reverting the
>   `componentDidUpdate` change makes it fail (the dark cursor comes back byte-identical to light).
> - `dark-mode filters the transparent fallback color, same as an explicit one` — pins the getter,
>   including that the no-theme paint path stays unfiltered.
>
> ```
> yarn vitest run packages/excalidraw/tests/bucketFill.test.tsx
> ```
>
> 33/33 pass. `yarn test:typecheck` clean. Also ran `interactivity`, `lasso` and `selection` —
> 151 pass, 1 skipped.

**Media, done:** `00-CURSOR-PROOF.gif` (6s loop) and `.mp4`, plus the static `.png`. All three are
built from the cursor SVG the editor actually installs, captured live from the dev build — not
mockups.

**Optional extra:** a true screen capture of the live OS cursor. Only worth it if a maintainer
asks; the GIF already shows the mismatch the issue is about. If you do record one:
open `localhost:3001` → pick the bucket-fill tool (`B`) → hover the canvas → `Alt+Shift+D` to
toggle dark → hover again. Windows: `Win+Alt+R`, and enable
*Settings → Accessibility → Mouse pointer* capture, or use ShareX with "capture cursor" on —
the default Game Bar recording omits the cursor, which is the whole subject.

---

## 2. Lane E — comment on #8067

`https://github.com/excalidraw/excalidraw/issues/8067` — *Keyboard shortcuts for changing stroke width*

> Is this still wanted? No open PRs on it as far as I can tell.
>
> The straightforward approach is to mirror the existing `increaseFontSize` / `decreaseFontSize`
> actions — same `keyTest` shape, same action registration, just targeting
> `currentItemStrokeWidth` and the selection.
>
> Two things I'd want a steer on before writing it:
>
> 1. I couldn't find an unclaimed chord. The obvious candidates collide with existing bindings, and
>    font size already uses the `Ctrl/Cmd + Shift + <` / `>` pair. Is there a combination you'd
>    prefer, or would you rather this stay unbound and be command-palette only?
> 2. Freedraw looks mid-rework (#11102). If stroke width is going to move as part of that, I'd
>    rather not add a shortcut on top of it right now.
>
> Happy to PR it once those are settled.

---

## 3. Lane E — comment on #11875

`https://github.com/excalidraw/excalidraw/issues/11875` — *cursor change when holding alt/option to indicate duplication*

> @<reporter> are you still working on this? Didn't want to duplicate effort — happy to leave it
> with you.
>
> If it's free: cursor state is centralised in `AppCursor` (`App.cursor.ts`) since the recent
> refactor, and this fits its existing shape well. `applyForTool()` sets the *resting* tool cursor
> and memoises it; hover and interaction affordances go through `cursor.set()`, which deliberately
> clears that memo so the next `applyForTool()` restores the resting cursor. So an Alt-held
> duplicate affordance would be a `cursor.set(copyCursor)` on the alt keydown while a moveable
> element is under the pointer, and a `cursor.reset()` on keyup or pointer-leave — no new state
> needed.
>
> The bit worth deciding first is scope: only while actually dragging, or also on plain hover with
> Alt held? Figma does the latter. Let me know which you'd want and I'll open a PR.

---

## 4. Lane C — comment on #11554

`https://github.com/excalidraw/excalidraw/pull/11554`

> Happy to draft the API docs for the unreleased viewport surface — `setViewport`,
> `getViewportOffsets`, `initialState.viewport`, plus the `interaction` / `activeTool` props from
> #11605 / #11665.
>
> Current state of `dev-docs`: `api/props/excalidraw-api.mdx` still lists `scrollToContent` in the
> API table and carries a full `## scrollToContent` section, and doesn't mention `setViewport` at
> all. `scrollToContent` also still appears in `initialdata.mdx` and
> `excalidraw-element-skeleton.mdx`. I realise the docs site tracks 0.18.1 and is therefore
> technically correct for npm users today, so I don't want to jump the gun.
>
> Gated however you like — behind an "unreleased" banner, or held on a branch until the next
> release. Just say which and I'll start.

**Note:** I verified this against master — `ExcalidrawImperativeAPI` exposes `setViewport` and
`getViewportOffsets` and no `scrollToContent`, and `app.scrollToContent` is `undefined` at runtime.
Re-check before posting in case it moved.

---

## 5. Also worth filing (not drafted as a comment yet)

The collab-room viewport finding from Nat's board: joining lands at 100% zoom on a scene spanning
~4,200 × 1,960 units, so a joiner sees a near-blank canvas until they manually zoom to fit (which
settles at 33%). Sits in the #11554 area. The Field Guide's Lane A route for this is *file it
yourself with a minimal scene + video*, not comment on an existing thread — so it needs a repro
recording before it's worth opening.
