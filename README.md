# Pursuit L3 — Excalidraw contribution work

Working documents for the L3 team's Excalidraw contribution sprint. This branch is
**docs only** — it carries no Excalidraw source and is never merged upstream. It exists so the
team has one place to read findings, evidence and staged text.

Team: Jason Zeng · Aisling Leiva-Davila · Mitra Kermanian · Natalie Walker

---

## Current priority

**[viewport-lock-issue.md](viewport-lock-issue.md)** — staged report for a new upstream issue:
a rigid scroll lock installed via `setViewport({ lock: { scroll: true } })` does not constrain
pointer-drag panning. The viewport leaves the box in every direction, repeatedly, with no snap-back.

Status: **not filed.** Blocked on a second-machine reproduction. Per the team's decision this is
**report only** — no PR — with "happy to take this" as the offer. Evidence is a 63-second screen
recording showing the lock installing and the viewport escaping in three directions.

Regression source: [excalidraw/excalidraw#11554](https://github.com/excalidraw/excalidraw/pull/11554).

---

## Contents

| Document | What it is |
|---|---|
| [viewport-lock-issue.md](viewport-lock-issue.md) | Staged issue text for the viewport-lock bug, plus pre-filing checklist |
| [excalidraw-full-audit.md](excalidraw-full-audit.md) | Repository and product audit of Excalidraw and excalidraw.com |
| [mitra-lane-a-findings.md](mitra-lane-a-findings.md) | Lane A QA findings and the theme-dependent cursor fix |
| [ready-to-post.md](ready-to-post.md) | Staged upstream comments and PR text, with per-item status |
| [team-guide-walkthrough.md](team-guide-walkthrough.md) | Walkthrough of the L3 team guide and board setup |
| `screenshots/` | Numbered evidence folders, one per investigation, plus QA findings JSON |

## Conventions

- Evidence goes in a numbered folder under `screenshots/`, named for the investigation
  (`01-baseline-bugs`, `06-lane-a-viewport-library`, …), and findings docs cite those paths.
- Nothing in `ready-to-post.md` or `viewport-lock-issue.md` has been posted upstream. Each carries
  its own status line — check it before assuming something was sent.
- Upstream work happens on separate branches off `master`, never on this one.
