# Ready to file — viewport lock does not constrain pointer-drag panning

**Target:** new issue on `excalidraw/excalidraw` (not a comment on #11554 — a new issue is trackable).
**Attachment:** `Screen Recording 2026-08-17 at 7.12.14 PM.mov` (63s — shows the Fit+LOCK button, the
install zoom, and escapes in three directions).
**Status:** not filed. Nothing below has been posted.

---

## Title

```
Viewport lock (setViewport lock.scroll) does not constrain pointer-drag panning
```

## Body

> ### Description
>
> With a rigid scroll lock installed via `setViewport({ lock: { scroll: true, overscroll: false } })`,
> the viewport can still be panned freely by dragging. The constrained box is left in every direction
> — up, left, and right — repeatedly, with no snap-back.
>
> Keyboard panning under the same lock *is* correctly constrained, which is what makes this look like
> a lock that works until you drag.
>
> ### Steps to reproduce
>
> 1. Install a rigid lock on a target box:
>    ```js
>    excalidrawAPI.setViewport({
>      target: [0, 0, 1000, 1000],
>      fit: "scale-down",
>      animation: false,
>      lock: { scroll: true, overscroll: false },
>    });
>    ```
> 2. Confirm the lock is installed (`appState.scrollConstraints` is non-null) and the viewport has
>    settled inside the box.
> 3. Drag-pan the canvas upward past the top edge of the box.
>
> ### Expected
>
> Panning stops at the box edge. With `overscroll: false` there should be no give at all.
>
> ### Actual
>
> The viewport leaves the box and stays there. Repeating the drag in any direction moves it further
> out. No snap-back occurs. In the attached recording the target rectangle leaves the screen entirely
> and stays gone for 8+ seconds.
>
> ### Where it looks like it comes from
>
> `constrainScrollState` itself appears correct — it is pure and well covered by unit tests in
> `packages/excalidraw/tests/scrollConstraints.test.tsx`. `AppViewport.translate` does apply it.
>
> The difference seems to be at the call sites. `translate` applies the clamp as a second,
> function-form `setState`:
>
> ```ts
> this.app.setState(state);
> this.app.setState((prevState) => {
>   ...
>   return constrainScrollState(prevState, overscroll);
> });
> ```
>
> The keyboard paths pass an **updater function**, so each step is computed from the latest queued
> (already clamped) state:
>
> ```ts
> // packages/excalidraw/components/App.tsx — PageUp/PageDown
> this.viewport.translate((state) => ({ scrollX: state.scrollX + offset }));
> ```
>
> The drag paths pass a **plain object** computed from `this.state`, which under throttled
> `pointermove` has not yet flushed the previous clamp:
>
> ```ts
> // App.tsx — handleCanvasPanUsingWheelOrSpaceDrag
> this.viewport.translate({
>   scrollX: this.state.scrollX - deltaX / this.state.zoom.value,
>   scrollY: this.state.scrollY - deltaY / this.state.zoom.value,
> });
>
> // App.tsx — handlePointerMoveOverScrollbars (both axes)
> this.viewport.translate({
>   scrollX: this.state.scrollX - (dx * multiplier) / this.state.zoom.value,
> });
> ```
>
> If that is the cause, each move re-bases from an unclamped snapshot and the clamp never compounds,
> which would explain why the escape is direction-agnostic and cumulative rather than an axis-specific
> quirk.
>
> I could not reproduce this in a jsdom test — `React.act()` flushes synchronously, so the batching
> window the bug appears to depend on does not exist there. That may also be why it wasn't caught:
> the only integration test for the lock covers keyboard panning
> (`"installs a scroll lock and prevents keyboard panning out of the box"`), and there is no
> integration coverage for drag-panning under a lock.
>
> Introduced by #11554.
>
> ### Environment
>
> - Reproduced on `master`
> - Chrome, Windows 11
>
> Happy to take this if you'd like a PR.

---

## Notes before filing (do not include in the issue)

- **Reproduce on a second machine first.** Two people, same result, is the standard this team agreed
  on and it makes the report unimpeachable.
- **Re-check `master` on the day you file** — `#11554` is in an area the maintainers are actively
  reworking, and a fix may have landed.
- **Do not open a PR unprompted.** The team's Field Guide lists viewport APIs as a hot zone
  (dwelle's SDK workstream, unreleased breaking changes). The "happy to take this" line is the ask;
  let a maintainer answer it. If dwelle says go, the four call sites above are the change.
- **The call-site analysis is a hypothesis**, not a proven mechanism — it is stated as such in the
  body on purpose. Overclaiming a root cause that turns out wrong costs more than the analysis gains.
- Attach the recording directly to the issue; GitHub accepts `.mov` up to 10 MB, otherwise convert
  to MP4 first.
