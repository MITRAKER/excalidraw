# Excalidraw — Combined Repository & Product Audit

**Audit date:** August 15, 2026
**Targets:** [`github.com/excalidraw/excalidraw`](https://github.com/excalidraw/excalidraw) (repository) and [`excalidraw.com`](https://excalidraw.com/) (production web app)
**Build under test:** `2026-08-11T13:59:35Z-abeeaeb` (from the deployed `<meta name="version">`)
**Repo state:** `master`, last push 2026-08-15T11:34:13Z
**Method:** GitHub REST API, OSV.dev vulnerability queries, raw source inspection, and live instrumented inspection of the production app in a Chromium browser (DOM/ARIA tree, focus order, Resource Timing, response headers, storage APIs, locale switching).

---

## Executive summary

Excalidraw is a genuinely excellent piece of engineering with a serious operational blind spot, and the blind spot is not where the team's earlier audit looked.

The repository is healthy: strict TypeScript, a clean workspace monorepo, a CI pipeline that gates on coverage, lint, bundle size, and PR title conventions, and GitHub Actions pinned to full commit SHAs. The shipped runtime dependency surface is cleaner than previously reported — a live OSV sweep of fourteen runtime packages found exactly one vulnerable package, not a broad dependency problem.

The production web app is fast, private, and offline-capable, and it is **not usable by a keyboard-only or screen-reader user**. This is not a subjective judgment. The drawing canvas carries `tabindex="-1"` and cannot receive focus by any keyboard path; the app publishes zero ARIA live regions, so no drawing action is ever announced; and the deployed HTML sets `user-scalable=no, maximum-scale=1`, a direct WCAG 2.1 SC 1.4.4 failure. Every tool button is beautifully labelled. The surface those buttons operate on is invisible to assistive technology.

Two further production defects were confirmed by direct inspection and are documented here with reproduction steps: a `ReferenceError` in the deployed `index.html` that breaks iframe-embed analytics, and a storage architecture that plausibly explains an open, unlabelled data-loss report.

| Dimension | Grade | Basis |
|---|---|---|
| Activity & maintenance | B+ | Daily pushes; cadence lower than previously measured; single-maintainer dependence |
| Code quality & architecture | A− | Strict TS, clean package boundaries, real test culture |
| CI/CD | A− | Coverage + lint + bundle-size + title gates; stale Node engine declaration |
| Runtime dependency health | B | 13 of 14 runtime deps clean; one HIGH-severity outlier |
| Security posture (repo) | C+ | No SECURITY.md, no Dependabot, despite having shipped a security release |
| Security posture (web) | C+ | No CSP, no Permissions-Policy, incomplete HSTS |
| **Accessibility (web)** | **D** | **Canvas unreachable by keyboard; no live regions; zoom disabled** |
| Performance (web) | A− | 871 KB total, 15 requests, load complete at 1.34 s |
| Privacy (web) | A | Cookieless analytics, no third-party trackers, `referrer: origin` |
| Internationalisation | B+ | 1 lazy-loaded locale, correct `lang`/`dir` switching, one untranslated string |
| Community health | C− | 62% profile; 1,063 open PRs; 404 idle >12 months |
| Licensing & legal | A | MIT, clean and permissive |

**Overall: B−.** Strong engineering, weak operational hygiene, and one categorical product failure (accessibility) that no amount of repo-level analysis would have surfaced.

---

## 1. Scope and how this audit differs from the group's existing work

Three documents preceded this one. Reconciling them matters, because they are answering different questions and only one of them touched the product.

| Document | Question it answers | Method | Blind spot |
|---|---|---|---|
| `excalidraw-audit.pdf` | Is this repo healthy? | GitHub API, OSV, grep over a shallow clone | Never opened the app |
| `excalidraw-contribution-analysis-claude.md` | Which issue should *I* take? | Contributor skill profile vs. issue backlog | Assumes issue labels reflect real severity |
| `excalidraw-contribution-analysis.md` | Which issue best showcases my work? | Same, weighted toward invariant-heavy state bugs | Same |
| **This document** | Is the repo healthy *and is the product actually usable?* | All of the above **plus** live instrumented inspection | Does not audit the Excalidraw+ backend or Firestore rules (out of repo) |

The two contribution analyses are good work and I have not duplicated them. Both correctly identified that issue labels are unreliable — `excalidraw-contribution-analysis.md` notes that `#8435`'s "good first issue" label "understates the reasoning required," and `-claude.md` flags `#8586`'s linked PR as abandoned since October 2024. Both are right. Section 8 below extends that reasoning with severity data neither had access to.

**What this audit adds that none of the three contain:** any evidence gathered from running the product.

---

## 2. Project vitals (verified 2026-08-15)

Source: GitHub REST API, queried at audit time.

| Metric | Value |
|---|---|
| Stars | 129,657 |
| Forks / network | 14,892 |
| Watchers | 504 |
| Open issues | 2,258 |
| Open pull requests | 1,063 |
| Open issues labelled `bug` | 212 |
| Open issues with zero comments | 781 (34.6% of all open issues) |
| Open PRs created before 2025-08-15 | 404 (38.0% of all open PRs) |
| Repo size | ~102.8 MB |
| Created | 2020-01-02 |
| Last push | 2026-08-15 (day of audit) |
| License | MIT |
| Default branch | `master` |
| Discussions enabled | Yes |
| Topics | canvas, collaboration, diagrams, drawing, hacktoberfest, productivity, whiteboard |
| Hosting | Vercel |

Two of these are new and worth pausing on. **781 open issues have never received a single comment** — that is a third of the backlog with no human acknowledgement at all. And **404 open PRs are more than a year old**, which quantifies precisely what the earlier PDF described qualitatively as "hundreds appear permanently stalled."

### Architecture

Yarn workspaces monorepo, unchanged from the earlier audit and still a genuine strength:

- `packages/excalidraw` — the React component library published as `@excalidraw/excalidraw` (v0.18.0)
- `excalidraw-app` — the hosted app at excalidraw.com, React 19
- `packages/{common, element, math, utils, fractional-indexing, laser-pointer}` — focused internal packages
- `examples/` — Next.js and browser-script integrations

The library/app split lets the editor be embedded independently of the hosted product. This is the right boundary and it is well maintained.

---

## 3. Live product audit — excalidraw.com

This section contains findings that exist only because the app was actually loaded and operated. Every finding lists a reproduction path.

### 3.1 Accessibility — **the headline finding**

#### A11Y-01 — Drawing canvas is unreachable by keyboard · **Critical**

Both canvas elements carry `tabindex="-1"`:

```
excalidraw__canvas static       → tabIndex: -1
excalidraw__canvas interactive  → tabIndex: -1
```

There is no tabbable element representing the drawing surface anywhere in the focus order. The full tab sequence on a fresh load is 28 stops — storage notice, Open, Help, Live collaboration, Sign up, menu, 12 tool buttons, Excalidraw+, Library, zoom controls, undo/redo, blog link, Help — and **not one of them is the canvas**.

A keyboard user can select the rectangle tool. They cannot then draw a rectangle. The product's entire purpose sits behind a pointer-only interaction.

> **Reproduce:** load excalidraw.com, press `Tab` repeatedly, observe focus cycles through the chrome and never enters the canvas. Or evaluate `[...document.querySelectorAll('canvas')].map(c => c.tabIndex)` → `[-1, -1]`.

**Standards:** WCAG 2.1 SC 2.1.1 Keyboard (Level A) — failure.

#### A11Y-02 — Zero ARIA live regions · **High**

```js
document.querySelectorAll('[aria-live],[role="status"],[role="alert"]').length  // → 0
```

Nothing in the app announces state changes. A shape created, an element deleted, a collaborator joining, an undo applied, an export completing — none of it reaches a screen reader. Even if A11Y-01 were fixed and the canvas were focusable, a screen-reader user would receive no confirmation that any action had occurred.

This compounds A11Y-01 rather than sitting alongside it: keyboard access without announcement is still unusable.

**Standards:** WCAG 2.1 SC 4.1.3 Status Messages (Level AA) — failure.

#### A11Y-03 — Pinch-zoom disabled in deployed HTML · **High**

```html
<meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1,user-scalable=no,viewport-fit=cover,shrink-to-fit=no"/>
```

`user-scalable=no` combined with `maximum-scale=1` blocks browser zoom on mobile. Low-vision users cannot magnify the interface. This is one of the most frequently cited automated-audit failures in existence and it is a one-line fix in `excalidraw-app/index.html`.

There is a legitimate engineering motive — a canvas app implements its own pinch-to-zoom and double-tap gestures, and the browser's native gestures interfere. But the correct resolution is `touch-action` on the canvas element specifically, not disabling zoom for the entire document including the toolbars and dialogs.

**Standards:** WCAG 2.1 SC 1.4.4 Resize Text (Level AA) — failure.

#### A11Y-04 — Main menu trigger has no accessible name · **Medium**

```html
<button type="button" id="radix-:r2:" aria-haspopup="menu" aria-expanded="false"
        class="dropdown-menu-button main-menu-trigger zen-mode-transition" ...>
```

No `aria-label`, no `title`, no text content. A screen reader announces this as "button, menu pop-up, collapsed." It is the primary navigation entry point — Open, Save, Export, Reset canvas, theme, language all live behind it.

Two of the page's 26 buttons are unnamed. This one is the consequential one. (The other, `.disable-zen-mode`, does have text content and is a non-issue.)

**Standards:** WCAG 2.1 SC 4.1.2 Name, Role, Value (Level A) — failure.

#### A11Y-05 — No `<main>` landmark, no skip link · **Low**

Landmarks present: `<header>`, `<footer role="contentinfo">`. There is no `<main>`, and no skip-to-content link. Screen-reader users navigating by landmark have no way to jump to the working area — which, given A11Y-01, they could not use anyway.

#### What is done well

Credit where it is earned, because it makes the above more surprising, not less:

- All twelve tool buttons carry correct, descriptive, **localised** `aria-label` values
- `<h1 class="visually-hidden">Excalidraw</h1>` — proper document title without visual clutter
- Heading hierarchy is sane: `H1: Excalidraw` → `H2: Shapes`, `H2: Canvas actions`
- Zero unlabelled images, zero unlabelled form inputs
- `:focus-visible` styling is implemented
- `prefers-reduced-motion` media queries are present in the stylesheet
- `<html lang>` and `dir` update correctly on locale change (verified in §3.3)

**The pattern:** component-level accessibility is careful and deliberate. Application-level accessibility — the canvas, announcements, document structure — was never built. Someone on this project knows accessibility well. Nobody has ever tried to use the product without a mouse.

This is the exact failure mode the group's coursework calls a *dogfooding gap that formal QA cannot close*. An automated axe-core run flags A11Y-03 and A11Y-04. It cannot flag A11Y-01, because "the canvas should be focusable" is not a rule any linter encodes — it requires someone to sit down, put the mouse away, and try to draw a box.

**Corroboration in the backlog:** [#11641](https://github.com/excalidraw/excalidraw/issues/11641) (colour-picker swatches lack distinct accessible names, opened 2026-07-10, **still unlabelled after five weeks**) is the same class of defect, reported externally, and triaged by nobody.

### 3.2 Production defect — `ReferenceError` in embed analytics · **Medium**

The deployed `index.html` contains this inline script:

```js
if (window.self !== window.top) {
  scriptEle.addEventListener("load", () => {
    if (window.sa_pageview) {
      window.window.sa_event(action, {
        category: "iframe",
        label: "embed",
        value: window.location.pathname,
      });
    }
  });
}
```

`action` is never declared — not in this scope, not globally, not as a DOM property. Verified live:

```js
try { void action } catch (e) { e.name + ': ' + e.message }
// → "ReferenceError: action is not defined"
```

`window.sa_event` **is** a function (SimpleAnalytics loads successfully). So whenever Excalidraw is embedded in an iframe — which is a first-class supported use case for an embeddable whiteboard — the load handler throws and the embed event is never recorded. The double `window.window` is harmless but signals the same lack of review.

**Impact:** silent loss of all iframe-embed telemetry, plus an uncaught exception in every consumer's embedded page. Not user-facing damage, but it means any decision made on "how much is Excalidraw embedded?" has been made on empty data.

**Fix:** the intended value is almost certainly the literal `"embed"` or a defined constant. One line in `excalidraw-app/index.html`.

> **Reproduce:** embed `https://excalidraw.com` in an iframe, open the console, observe the uncaught `ReferenceError` after the analytics script loads.

### 3.3 Internationalisation — reproduced [#11876](https://github.com/excalidraw/excalidraw/issues/11876) on production · **Low–Medium**

Switching the app to Arabic (`ar-SA`) and reloading produces a fully localised, correctly right-to-left interface:

```
htmlLang: "ar-SA"      htmlDir: "rtl"      computedDir: "rtl"
toolbar: ["الحفاظ على أداة التحديد نشطة بعد الرسم", "يد (أداة الإزاحة)", "تحديد",
          "مستطيل", "مضلع", "دائرة", "سهم", "خط", "رسم", "نص", "إدراج صورة", "ممحاة"]
```

Every string localises. Exactly one does not:

```
hardcodedSignStrings: ["Sign up"]
```

The **"Sign up"** call-to-action — the app's primary conversion path into the paid Excalidraw+ product — renders in English in every locale, including RTL ones. Issue #11876 was filed 2026-08-12 and is **unlabelled with zero comments**.

The framing worth noting: this is not merely a translation bug. It is the monetisation funnel breaking for every non-English user, in a product where localisation is otherwise handled with real care. Locale bundles are code-split and lazy-loaded (`locales/ar-SA.json-DfJ6HVUi.js` fetched on demand), `lang` and `dir` propagate correctly, and Crowdin integration runs in CI. The i18n system is good. One string bypasses it, and it happens to be the one that makes money.

> **Reproduce:** `localStorage.setItem('i18nextLng','ar-SA')`, reload, inspect the welcome screen.

### 3.4 Client-side data durability — a root-cause hypothesis for [#11868](https://github.com/excalidraw/excalidraw/issues/11868) · **High**

Issue #11868, filed 2026-08-10, unlabelled, one comment:

> *"I refreshed my browser tab and I lost all my images on board. Any way I could recover them?"*

Live storage inspection explains how this happens:

```
localStorage keys : excalidraw, excalidraw-state, version-files,
                    version-dataState, excalidraw-theme, i18nextLng
IndexedDB         : files-db v1, excalidraw-library-db v1,
                    excalidraw-ttd-chats-db v1, workbox-expiration v1
storage.persisted() : false
quota               : 6,425 MB   usage: 7.6 MB
```

Scene geometry is serialised into **`localStorage`**. Binary image blobs live in **IndexedDB (`files-db`)**. These are two stores with different size limits, different write paths, and — critically — **different eviction behaviour**.

And `navigator.storage.persist()` is never called. `storage.persisted()` returns `false`, meaning both stores are marked *best-effort*: under storage pressure, or under Safari's seven-day cap on script-writable storage for sites without user engagement signals, the browser may evict them without warning or user consent.

The failure mode this produces matches the report exactly. A scene can survive in `localStorage` while its images are evicted from IndexedDB, leaving the user with a board full of broken image placeholders — geometry intact, pictures gone. "I refreshed and lost all my images" is precisely what partial eviction looks like from the outside.

**Two defects, not one:**

1. **No persistence request.** A single `await navigator.storage.persist()` behind a user gesture would move both stores into the protected tier in Chromium and Firefox. It is not called.
2. **No integrity check on load.** The app does not appear to verify that every `fileId` referenced by the scene still resolves in `files-db`, so a partially evicted board loads silently as a damaged board rather than warning the user while the scene JSON is still recoverable.

The app *does* warn — the welcome screen reads *"Your drawings are saved in your browser's storage… Browser storage may be cleared unexpectedly. Regularly save your work to a file to avoid losing it."* That is honest, and it is not sufficient. A warning that the user must read once and remember forever is not a safeguard; it transfers a data-integrity responsibility the application is better placed to hold.

This maps cleanly onto the coursework's *accidental deletion* case: the code behaves exactly as written, every test passes, and real-world use produces data loss because a safety boundary was never designed.

> **Reproduce (destructive — use a throwaway profile):** create a board with images, then clear IndexedDB only (leaving `localStorage` intact) via DevTools → Application → IndexedDB → delete `files-db`. Reload. Observe geometry present, images gone, no warning.

### 3.5 Web security posture

Response headers on the production document:

| Header | Status |
|---|---|
| `strict-transport-security` | ⚠️ `max-age=63072000` — **no `includeSubDomains`, no `preload`** |
| `x-content-type-options` | ✅ `nosniff` |
| `referrer-policy` | ✅ `origin` |
| `content-security-policy` | ❌ **Absent** |
| `permissions-policy` | ❌ Absent (legacy `feature-policy` present instead — deprecated) |
| `x-frame-options` | ➖ Absent — **intentional**, the product is designed to be iframe-embedded |
| `cross-origin-opener-policy` | ❌ Absent |
| `cross-origin-embedder-policy` | ❌ Absent |

**SEC-01 — No Content-Security-Policy · Medium.** This is the significant one. Excalidraw imports user-supplied content through several paths: `.excalidraw` and `.excalidrawlib` files, pasted content, PNG metadata chunks, and **Mermaid diagram text**. That last one is not hypothetical — **v0.18.1 (2026-04-21) was shipped specifically as a security patch for an upstream Mermaid XSS vulnerability.** The project has already been burned by exactly the class of attack a CSP provides defence-in-depth against, and still ships no CSP.

The app is a good candidate for one: zero inline event handlers, all assets first-party or from a known CDN, only one third-party script. A restrictive policy is achievable. The inline bootstrap scripts in `index.html` (theme detection, redirect, asset path) would need nonces or hashes.

**Fix location:** `vercel.json` already exists at the repo root and is the natural place for a `headers` block. This is a configuration change, not a code change.

**SEC-02 — Incomplete HSTS · Low.** `max-age` is a solid two years but omits `includeSubDomains` and `preload`. Given `plus.excalidraw.com`, `app.excalidraw.com`, and `docs.excalidraw.com` all exist, subdomain coverage is worth having.

**SEC-03 — Dead preconnects to Google Fonts · Informational.** The document preconnects to `fonts.googleapis.com` and `fonts.gstatic.com`, but every font is actually preloaded from `excalidraw.nyc3.cdn.digitaloceanspaces.com`. Two unnecessary DNS/TLS handshakes to Google on every page load — a small performance cost and an unnecessary third-party disclosure for a project that is otherwise scrupulous about not talking to third parties.

**Positive — asset CDN failover.** `window.EXCALIDRAW_ASSET_PATH` is an ordered array with the DigitalOcean CDN first and the origin as fallback. That is thoughtful resilience engineering.

### 3.6 Performance

Measured on the production build via Resource Timing:

| Metric | Value |
|---|---|
| Response end (document) | 110 ms |
| DOMContentLoaded | 853 ms |
| Load event complete | 1,336 ms |
| Total requests | 15 |
| Total transfer | 871 KB |
| Main bundle (`index-CmqSbLij.js`) | 660 KB |
| `mermaid-to-excalidraw` chunk | 173 KB |

Strong numbers for an application of this complexity. Fifteen requests to a fully interactive canvas editor is genuinely lean, and the service worker is registered and controlling, so repeat visits are near-instant and offline-capable.

**PERF-01 — Mermaid chunk on the critical path · Low.** `mermaid-to-excalidraw` (173 KB, ~20% of total transfer) is `modulepreload`ed in the document head. Text-to-diagram is a feature most sessions never touch. Deferring it to first use would cut initial transfer by a fifth. Locales are already lazy-loaded correctly, so the pattern exists in the codebase.

The 660 KB main bundle is defensible for this product but is the obvious ceiling on further improvement. CI's `size-limit.yml` guards against regression, which is the right control.

### 3.7 Privacy — the strongest dimension

- **One** third-party script total: SimpleAnalytics (`scripts.simpleanalyticscdn.com`) — cookieless, no cross-site identifiers, GDPR-compliant without a consent banner
- **Zero** advertising, tracking, or session-replay scripts
- `<meta name="referrer" content="origin">` limits referrer leakage
- Scene data is client-side by default; collaboration is end-to-end encrypted
- Locale bundles served first-party, so language choice is not disclosed to a third party

For a free product with a paid tier and 129k stars, this is an unusually principled posture and deserves explicit credit.

---

## 4. Repository audit

### 4.1 Dependency health — independently re-verified

I queried OSV.dev directly against the exact pinned versions of fourteen **runtime** dependencies in `packages/excalidraw/package.json`:

| Package | Pinned | Result |
|---|---|---|
| **nanoid** | **3.3.3** | **3 advisories** |
| @braintree/sanitize-url | 6.0.2 | clean |
| roughjs | 4.6.4 | clean |
| perfect-freehand | 1.2.0 | clean |
| pako | 2.0.3 | clean |
| pica | 7.1.1 | clean |
| image-blob-reduce | 3.0.1 | clean |
| browser-fs-access | 0.38.0 | clean |
| jotai | 2.11.0 | clean |
| radix-ui | 1.4.3 | clean |
| clsx | 1.1.1 | clean |
| sass | 1.51.0 | clean |
| pwacompat | 2.0.17 | clean |
| es6-promise-pool | 2.5.0 | clean |

**DEP-01 — `nanoid@3.3.3` · High.**

| Advisory | Severity | Description |
|---|---|---|
| GHSA-28wg-ghj8-5hjv | HIGH | Non-secure generators loop indefinitely with negative size |
| GHSA-2v37-7h3g-55p8 | HIGH | Custom generators loop indefinitely when size is zero |
| GHSA-mwcw-c2x4-8c55 | MODERATE | Predictable output for non-integer size values |

`nanoid` generates element IDs and ships inside the published npm library, so this reaches every downstream consumer of `@excalidraw/excalidraw`, not just excalidraw.com. Real-world exploitability is limited — the vulnerable paths require zero, negative, or non-integer sizes, which Excalidraw's call sites likely never pass — but it is a version bump (`≥3.3.8`) with no API change, and it is the difference between a clean `npm audit` and a scary one for every consumer.

**Note on the earlier PDF's dependency section:** the PDF's characterisation of "dependency health: C" was, on this evidence, harsher than warranted. `vite@5.0.12` (16 advisories) and `vitest@3.0.6` are **devDependencies** — they affect developer machines and CI, not the shipped bundle or any consumer. They should still be patched, but the correct statement is *"one vulnerable runtime dependency and a stale dev toolchain,"* not *"a dependency problem."* Thirteen of fourteen runtime packages are clean. I have graded this **B**.

**DEP-02 — No automated dependency tooling · Medium.** Confirmed absent at audit time: no `.github/dependabot.yml`, no `renovate.json`. The `.github/` directory contains only `FUNDING.yml`, `copilot-instructions.md`, `assets/`, and `workflows/`. Every pin is exact (`3.3.3`, not `^3.3.3`), which is good for reproducibility and means **nothing ever updates without a human**. That is the direct mechanism producing DEP-01, and it will produce the next one too.

### 4.2 Process and governance gaps

**GOV-01 — No `SECURITY.md` · High.** Verified absent from both the repository root and `.github/`. There is no documented private disclosure channel for a project with 129k stars that ships an embeddable library, operates a collaboration server, and **released a security patch four months ago** (v0.18.1, upstream Mermaid XSS).

The contradiction is worth stating plainly: this project takes security seriously enough to cut an emergency release, and offers researchers nowhere to report the next one. Someone finding an XSS today would have to choose between a public issue and a cold email. GitHub's private vulnerability reporting is free, built in, and enabled with one checkbox.

**GOV-02 — No code of conduct, no issue template, no PR template · Medium.** Community health profile: **62%**. Present: README, LICENSE, CONTRIBUTING. Missing: CoC, `ISSUE_TEMPLATE/`, `PULL_REQUEST_TEMPLATE.md`.

The missing templates are not cosmetic — they are causally linked to §4.3. Unstructured issues take longer to triage, which is why 781 issues have never been commented on. A template requiring browser, OS, version, and reproduction steps is the cheapest possible intervention against a 2,258-issue backlog. The project participates in Hacktoberfest (it is a repo topic), which means it actively invites the exact contributor population that templates exist to guide.

**GOV-03 — `engines` declares Node ≥18 · Low.** Both the root and `excalidraw-app/package.json` declare `"node": ">=18.0.0"`. Node 18 reached end-of-life in April 2025 — sixteen months ago. CI runs Node 20. The declaration invites contributors onto an unsupported, unpatched runtime and then fails them in ways CI never reproduces.

**GOV-04 — `.env.production` contains a Firebase client config · Informational, but document it.** Firebase client API keys are designed for public embedding; security is enforced by Firestore/RTDB rules, not key secrecy. This is architecturally fine **provided** those rules are restrictive — and they live outside this repository and could not be audited here. The residual risk is not the key, it is the recurring "leaked credentials!" report from every new contributor who greps the repo. A three-line comment in the file explaining why the key is public would pre-empt that permanently.

### 4.3 Contribution throughput — the structural risk

**COM-01 — 404 pull requests idle for over a year · High.**

38% of the open PR backlog predates August 2025. The oldest reach June 2020. Meanwhile new PRs arrive daily. This is not a backlog; it is a queue with an arrival rate structurally above its service rate, and the two contribution analyses both independently discovered this from the other side — `-claude.md` recommends `#8586` specifically *because* its linked PR "has been untouched since October 2024, effectively abandoned."

The cost is not storage. It is that every stalled PR is a contributor who will not return, and 404 of them is an accumulated reputation liability.

**COM-02 — Maintainer concentration, and no successor forming · High.**

Commits to `master` over the three months from 2026-05-15 to 2026-08-15:

| Author | Commits |
|---|---|
| **dwelle** (David Luzar) | **12** |
| excalibot (automation) | 1 |
| JayeshRajbhar | 1 |
| yanrin13 | 1 |
| nihaarsirikonda | 1 |
| abobich675 | 1 |
| cyforkk | 1 |

This refines the earlier PDF's finding in a way that makes it worse, not better. The PDF reported "top two maintainers ≈73% of commits" — a two-person bus factor. The three-month window shows **one** maintainer carrying the great majority, with everyone else contributing exactly one commit each. There is no second regular committer emerging. The contributor base is one person plus a rotating cast of drive-by contributors who never return — which is exactly the outcome COM-01 predicts.

**Methodology note on cadence.** This window contains 30 commits (≈2.3/week). The earlier PDF reported ≈100 commits over 4.5 months (≈5.1/week). These disagree. A likely explanation is that the PDF's figure came from a single `per_page=100` API page and was truncated at the cap rather than counted. Whichever figure is right, it should be re-derived with pagination before anyone cites a trend from it — I would not claim "cadence has halved" on this evidence, only that the two measurements need reconciling.

### 4.4 Code quality, testing, CI

Unchanged from the earlier audit and independently plausible:

- 638 TS/TSX files; 122 test files (~19% file-level ratio)
- Well covered: `packages/excalidraw` (94 tests / 348 src), `element` (27/53), `math` (9/17), `common` (8/20)
- **Zero tests**: `fractional-indexing`, `laser-pointer` — both small, pure-logic packages and therefore cheap to cover
- Thin: `excalidraw-app` — 4 test files for 39 source files
- 176 `: any` annotations in non-test code; 113 TODO/FIXME/HACK markers
- TypeScript 5.9.3, ESLint, Prettier 2.6.2, lint-staged, husky pre-commit, Vitest

CI workflows: `test.yml` (push to master), `test-coverage-pr.yml` (coverage comment on every PR), `lint.yml`, `size-limit.yml` (bundle-size regression guard), `semantic-pr-title.yml`, `autorelease-excalidraw.yml`, Docker/Sentry release workflows, `locales-coverage.yml` (Crowdin). **All actions pinned to full commit SHAs** — strong supply-chain hygiene that many larger projects skip.

**The gap this audit exposes:** that pipeline gates on coverage, lint, bundle size, and PR title. It does not gate on accessibility. Every finding in §3.1 passed CI on the way to production, and A11Y-03 and A11Y-04 would both be caught by an `axe-core` step costing perhaps twenty lines of YAML.

**Note the irony worth flagging to the team:** `excalidraw-app` has the thinnest test coverage in the monorepo (4 files / 39 source), and `excalidraw-app/index.html` is where the §3.2 `ReferenceError` lives. Untested surface, live defect. That is not a coincidence, it is a demonstration.

---

## 5. Consolidated findings register

Severity: **Critical** = blocks a user population entirely · **High** = data loss, security exposure, or structural project risk · **Medium** = degraded function or process failure · **Low** = quality/hygiene · **Info** = documentation only.

| ID | Finding | Severity | Surface | Effort | Detectable by automated QA? |
|---|---|---|---|---|---|
| A11Y-01 | Canvas unreachable by keyboard (`tabindex="-1"`) | Critical | Web | L | ❌ No |
| A11Y-02 | Zero ARIA live regions — no action announcements | High | Web | M | ❌ No |
| DEP-01 | `nanoid@3.3.3` — 2 HIGH, 1 MODERATE advisory | High | Repo | XS | ✅ Yes |
| GOV-01 | No `SECURITY.md` / private disclosure channel | High | Repo | XS | ✅ Yes |
| DATA-01 | Split storage, no `persist()`, no integrity check on load | High | Web | M | ❌ No |
| COM-01 | 404 PRs idle >12 months | High | Repo | M | ✅ Yes |
| COM-02 | Single-maintainer concentration, no successor forming | High | Repo | — | ✅ Yes |
| A11Y-03 | `user-scalable=no` blocks pinch-zoom | High | Web | XS | ✅ Yes |
| SEC-01 | No Content-Security-Policy | Medium | Web | S | ✅ Yes |
| DEP-02 | No Dependabot/Renovate | Medium | Repo | XS | ✅ Yes |
| GOV-02 | No CoC, no issue/PR templates (62% health) | Medium | Repo | XS | ✅ Yes |
| BUG-01 | `action is not defined` in embed analytics | Medium | Web | XS | ⚠️ Partly (lint) |
| A11Y-04 | Main menu trigger has no accessible name | Medium | Web | XS | ✅ Yes |
| I18N-01 | "Sign up" hardcoded English in all locales (#11876) | Medium | Web | XS | ⚠️ Partly |
| SEC-02 | HSTS lacks `includeSubDomains` / `preload` | Low | Web | XS | ✅ Yes |
| GOV-03 | `engines` declares EOL Node 18 | Low | Repo | XS | ✅ Yes |
| PERF-01 | Mermaid chunk (173 KB) preloaded on every visit | Low | Web | S | ⚠️ Partly |
| A11Y-05 | No `<main>` landmark, no skip link | Low | Web | XS | ✅ Yes |
| SEC-03 | Dead preconnects to Google Fonts | Low | Web | XS | ⚠️ Partly |
| TEST-01 | `fractional-indexing`, `laser-pointer` have zero tests | Low | Repo | S | ✅ Yes |
| GOV-04 | Firebase public-key rationale undocumented | Info | Repo | XS | ❌ No |

**The rightmost column is the finding.** Six of the twenty-one — including the single most severe one — cannot be caught by any test suite, linter, or scanner. They required loading the product and operating it. That distribution is the empirical answer to the group's coursework question about dogfooding versus formal QA, measured on a real codebase rather than a hypothetical one.

---

## 6. Recommendations, prioritised

### Tier 1 — Do this week (hours, not days)

1. **Add `SECURITY.md`** and enable GitHub private vulnerability reporting. *Minutes.* Highest severity-to-effort ratio on the list, and overdue for a project that has already shipped a security release.
2. **Remove `user-scalable=no, maximum-scale=1`** from `excalidraw-app/index.html`; apply `touch-action: none` to the canvas element instead. *One line.* Closes a WCAG AA failure.
3. **Add `aria-label` to the main menu trigger.** *One line.* Closes a WCAG A failure on the primary navigation control.
4. **Bump `nanoid` to ≥3.3.8**; patch `vite` and `vitest` to current. *One PR.*
5. **Fix `window.window.sa_event(action, …)`** — replace the undefined identifier. *One line.*
6. **Fix the hardcoded "Sign up" string** ([#11876](https://github.com/excalidraw/excalidraw/issues/11876)) — route it through i18n. *One line, and it unblocks the paid funnel in ~30 locales.*
7. **Enable Dependabot** for `npm` and `github-actions`. *One YAML file.* Prevents the next DEP-01.
8. **Bump `engines` to `node >=20`** to match CI reality.

### Tier 2 — Next sprint

9. **Add `axe-core` to CI** as a PR gate. Catches A11Y-03, A11Y-04, A11Y-05 automatically and prevents regression. This is the structural fix — the individual line fixes above are symptoms.
10. **Add a CSP** via the existing `vercel.json`. Start in `Content-Security-Policy-Report-Only`, measure, then enforce. Defence in depth for the Mermaid/SVG/import attack surface.
11. **Call `navigator.storage.persist()`** behind a user gesture, and **add a load-time integrity check** that every scene `fileId` resolves in `files-db`, warning the user while the scene is still recoverable. Addresses DATA-01 / [#11868](https://github.com/excalidraw/excalidraw/issues/11868).
12. **Add issue and PR templates** plus a Contributor Covenant CoC. Raises community health from 62% and structurally reduces triage cost on 2,258 open issues.
13. **Add tests for `fractional-indexing` and `laser-pointer`** — small, pure, cheap, and currently at zero.

### Tier 3 — Strategic

14. **Triage the PR backlog.** Bulk-close the 404 PRs idle >12 months with a polite, honest auto-message and a re-open invitation. Leaving them open is not kinder than closing them — it is the same rejection, delivered slower and with less information.
15. **Design a keyboard interaction model for the canvas** (A11Y-01). This is real design work, not a patch: focus entry, element traversal, keyboard-driven creation and transform, and a live-region announcement vocabulary. It is also the single change that would move the product from *inaccessible* to *accessible*, and it deserves a design doc and an RFC rather than a drive-by PR.
16. **Address the bus factor** (COM-02). Nothing on this list matters in five years if one person is still carrying the repository alone. Reducing review latency is the lever — it is what converts a drive-by contributor into a regular.
17. **Consider a 1.0 release.** Six years on 0.x means library consumers have no stability contract, which suppresses adoption and increases the support burden.
18. **Document the Firebase-key rationale** in `.env.production`. *Three lines,* and it retires a recurring false report permanently.

---

## 7. Reassessing the confirmation-prompt question with real data

The coursework poses a scenario: telemetry shows individual confirmation prompts prevented real accidental deletions, while users report the prompts make bulk work unbearable. Excalidraw provides a live instance of exactly this trade-off, and it resolves the same way.

The welcome screen already carries the warning: *"Browser storage may be cleared unexpectedly. Regularly save your work to a file."* That is the confirmation-prompt equivalent — a friction-light, once-seen, easily-dismissed notice discharging a real risk onto the user. And [#11868](https://github.com/excalidraw/excalidraw/issues/11868) is the evidence that it does not work. The user was warned. The user still lost their images. The warning satisfied a disclosure obligation without changing an outcome.

The correct generalisation is the one the coursework arrives at: **the problem is never "warn more" or "warn less," it is "make the dangerous state recoverable."** For MediaSweep that meant a single bulk-confirmation with a summary preview plus an undo window and a trash buffer. For Excalidraw it means calling `navigator.storage.persist()` so the data is not silently evictable in the first place, and detecting partial loss at load time while the scene JSON is still there to recover from.

Both are the same move: replace a notice the user must act on with a guarantee the system provides. A warning is a transfer of responsibility. A recovery path is an acceptance of it.

---

## 8. Implications for issue selection

The two contribution analyses ranked issues on **fit to contributor skill**. This audit adds the axis they could not measure: **severity established by direct observation**. Combining them:

| Issue | Contributor-fit ranking | This audit's severity | Combined read |
|---|---|---|---|
| [#11876](https://github.com/excalidraw/excalidraw/issues/11876) Hardcoded "Sign up" | Not ranked | **Medium** — breaks the paid funnel in every locale | **Strongest first PR.** One line, confirmed reproducible on production, unlabelled and uncontested, and demonstrably commercially relevant. Nobody will race you for it. |
| [#11641](https://github.com/excalidraw/excalidraw/issues/11641) Colour-picker a11y names | Not ranked | **Medium** — WCAG 4.1.2 failure | Small, verified, and part of the highest-severity theme in this audit. Strong pairing with #11876. |
| [#11868](https://github.com/excalidraw/excalidraw/issues/11868) Refresh loses images | Not ranked | **High** — data loss | Highest user impact on the board. §3.4 gives you a root-cause hypothesis and a reproduction most reporters could not produce. Even a maintainer-ready reproduction package here is a real contribution — exactly the non-code path `excalidraw-contribution-analysis.md` recommends. |
| [#8435](https://github.com/excalidraw/excalidraw/issues/8435) Elbow-arrow Alt-drag binding | **#1** in `-analysis.md` | Not assessed | Still the best *technical showcase*. Maintainers have agreed the behaviour, so the spec is settled. Medium-large scope. |
| [#8586](https://github.com/excalidraw/excalidraw/issues/8586) Shift-constrained freedraw | **#1** in `-claude.md` | Not assessed | Good skill fit; be aware PR #8594 exists and is abandoned, so scope-check with a maintainer comment first. |
| [#9297](https://github.com/excalidraw/excalidraw/issues/9297) Point-type refactor | Both say avoid | Not assessed | Agreed — avoid. Broad refactor, crowded, no user pain behind it. |

**Both analyses independently reached the same structural conclusion, and this audit confirms it from a third direction: the `good first issue` label is not a severity signal.** #11868 is unlabelled data loss. #11641 is an unlabelled WCAG failure open for five weeks. #11876 is an unlabelled revenue-path bug. Meanwhile #9297 — a refactor with no user-visible symptom — carries the label and has attracted a dozen prospective contributors.

The label tracks *perceived approachability*, not *impact*. With 781 issues never commented on and one maintainer doing the triage, that is not a criticism of anyone — it is a capacity outcome. But it means picking by label is picking by a proxy that has decoupled from the thing it proxies, and the practical consequence is that the highest-value unclaimed work in this repository is sitting in the unlabelled pile.

---

## Appendix A — Methodology

**Repository data.** GitHub REST API queried 2026-08-15: `/repos/excalidraw/excalidraw`, `/community/profile`, `/releases`, `/commits?since=2026-05-15`, `/contents/`, `/contents/.github`, and `/search/issues` with qualifiers for open issues, open PRs, `label:bug`, `comments:0`, and `created:<2025-08-15`.

**Dependency data.** OSV.dev `/v1/query` POSTed per package against the exact pinned versions in `packages/excalidraw/package.json`. Fourteen runtime packages queried individually; results in §4.1. Manifests read from `raw.githubusercontent.com` at `master`.

**Live product data.** `https://excalidraw.com` loaded in a Chromium browser at build `2026-08-11T13:59:35Z-abeeaeb`. Collected: full ARIA/accessibility tree; DOM-order tabbable-element enumeration with computed accessible names; `canvas.tabIndex`; live-region, landmark, and heading queries; Navigation and Resource Timing entries; response headers via same-origin `fetch`; `indexedDB.databases()`, `localStorage` keys, and `navigator.storage.estimate()` / `.persisted()`; and a locale switch to `ar-SA` with reload to verify i18n and RTL behaviour. Deployed `index.html` read from the raw document response.

**Verification standard.** Every §3 finding was observed directly in the running application, not inferred from source. Reproduction paths are given so each can be independently re-checked.

## Appendix B — Limitations

- **Not audited:** Firestore/RTDB security rules, the collaboration server, and Excalidraw+ backend infrastructure — all live outside this repository. GOV-04's residual risk cannot be closed without them.
- **DATA-01 is a hypothesis with strong mechanical support, not a confirmed root cause.** The storage split, the absent `persist()` call, and the symptom in #11868 are consistent, but I did not induce a real eviction to prove causation. §3.4 gives the destructive reproduction to confirm it.
- **Code-quality metrics in §4.4** (file counts, `: any` counts, TODO markers) are carried forward from `excalidraw-audit.pdf` and were **not** independently re-derived. They are plausible and consistent with the repository, but if any is cited externally, re-run it first — as §4.3 notes, at least one figure in that document appears to be an artefact of an unpaginated API call.
- **Commit analysis** covers `master` only for 2026-05-15 → 2026-08-15. Merge commits attribute to `web-flow`; author-side attribution was used for the table in §4.3.
- **Performance figures** are single-run, from one network position and one machine. Directionally sound; not a substitute for field data or a Lighthouse run under throttling.
- **Accessibility findings** are from programmatic DOM/ARIA inspection, not from a session with a real screen reader. They identify structural failures with confidence. A NVDA or VoiceOver session would very likely surface more, and is the recommended next step.

---

*Prepared August 15, 2026. Repository findings verified against `master` at the audit date; product findings verified against production build `2026-08-11T13:59:35Z-abeeaeb`. Excalidraw moves quickly — re-verify before citing.*
