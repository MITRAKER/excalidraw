# Team Guide Walkthrough — prepared for Mitra

Source: `20260807_L3 team guide_ Getting to know your team.docx`
Board: `excalidraw.com/#room=2a3099494f40cfeb7dbf,…` (Nat's — **Team Charter v0**)
Team: Jason Zeng · Aisling Leiva-Davila · **Mitra Kermanian** · Natalie Walker

**Nothing here has been written to the board or posted anywhere.** Parts 1–5 are explicitly a
live team activity — the guide says *"Be honest in this session, not polite."* I can't supply
your teammates' answers, and I won't invent yours. What follows is: current status, the fixes the
board needs, what you personally should walk in ready to say, and the parts I *can* fully draft.

---

## Status at a glance

| Part | Deliverable | Status |
|---|---|---|
| Setup | 5 labelled zones on a shared board | 🟡 built, **one zone mislabelled** |
| Setup | Per-teammate colour assignment | 🟡 4 colours placed, **none labelled with a name** |
| 1 | Working & Communication Style | ❌ empty |
| 2 | Communication Norms + Slack channel + calendar invites | ❌ empty |
| 3 | Dividing Work | ❌ empty |
| 4 | Handling Conflict | ❌ empty — **and the zone is labelled "Handling work"** |
| 5 | Our 3 Commitments | ❌ empty |
| 6 | Trello board | ❌ no evidence it exists |

---

## Board fixes needed before the session

1. **Rename "Handling work" → "Handling Conflict".** Part 4 is conflict *and disengagement*.
   Left as-is, the zone collects workload logistics that Part 3 already covers, and the
   disengagement plan — the thing the guide says teams most often skip entirely — never gets
   written. This is Nat's board and it's live; she should make the change.
2. **Label the four colour swatches with names.** They're the right idea (4 shapes, 4 people) but
   unlabelled, which defeats the stated purpose: *"anyone looking at the board later can see who
   said what, without having to ask."*
3. **Delete the stray zero-size freedraw** at `(1494, -666)` in the Communication Norms zone.

---

## Part 1 — Working & Communication Style

Nine questions across working style, communication style, and skill/comfort. Yours to answer
honestly; I've only filled in what's actually evidenced.

**What the record shows about you** (from the Field Guide's audit of your public repos):

- Ship in **Vite + React + TS**; `Fashion_AI_Pilot` (zero-dep Node API, three.js showroom with
  cloth sim and postprocessing), `AmplifiAI` (Next 16 / React 19 / Tailwind 4 / Supabase)
- **7 merged PRs into peer repos** — you already work in other people's codebases
- **Conventional commits** (`fix(frontend): …`) — you match repo conventions without being asked
- Python FastAPI backend work on Digi-Child

**Prompts you need your own answer for** — don't let me guess these:

- Best focused work: mornings / evenings / quiet / music?
- Plan first, or start and figure it out?
- Stuck: want to be asked, or left to try first?
- Feedback: blunt, or context first?
- Overwhelmed: will you actually say so, or go quiet? What would make saying so easier?
- Realistic availability outside class — the guide says *realistically, not aspirationally*

### The skill-gap question — the one the guide says not to skip

> *"Is there a real difference in comfort level on this team? Name it now, honestly."*
> *"How will your team make sure the most experienced person doesn't just end up doing
> everything, and the least experienced person doesn't end up sitting out?"*

There is a real difference, and it's documented:

| | Canvas/2D | Testing | a11y | CI |
|---|---|---|---|---|
| Jason | ●●● | ●●● | ●●● | strong (90% coverage gate, 3-OS matrices) |
| Aisling | ●○○ (SVG) | ●○○ | ●●● | strong |
| **Mitra** | ●○○ (three.js 3D ●●) | ●○○ | ●○○ | **none anywhere** |
| Natalie | ●○○ (data viz) | ●○○ | ●○○ | Playwright/CI exposure |

**Lane A pairs you with Jason as co-fixer.** He is 3/3 on both canvas and testing; you're 1/3 on
canvas 2D with no CI history. That is exactly the "most experienced person does everything"
setup, and it's worth naming on the board rather than discovering in week 2.

Two things worth saying in your own defence, because the matrix undersells you: three.js is
harder maths than 2D canvas and transfers, and 7 merged outside PRs is the single most relevant
signal for open-source contribution — more than any framework rating on that table.

**Concrete proposal to put in the zone:** Jason and Mitra do not split Lane A by difficulty.
Split by *area*, and whoever takes an area writes its regression test. That prevents the
experienced person absorbing the hard half by default.

---

## Part 2 — Communication Norms

Requires actual actions in the session, not just stickies:

- [ ] Create a dedicated team Slack channel; **add all four people before leaving the session**
- [ ] Send real calendar invites for any standing time — the guide is pointed that
      *"a time nobody put on their calendar is a time that gets missed"*

Then three stickies: what belongs in-channel vs DM vs live; response time in and out of class
hours; whether `@channel` is fair game or reserved for urgent.

**Worth raising given this team:** the Field Guide's whole strategy is *"find the bug ourselves,
file it clean, fix it the same day"* — maintainers merge outsider PRs in 1–2 days, and the tracker
is saturated with duplicate PRs. That workflow only holds if a find reaches a fixer within hours.
Propose an explicit norm: **a Lane A find gets posted in-channel immediately, and the fixer claims
it in-channel before starting**, so two of you never fix the same thing.

---

## Part 3 — Dividing Work

The guide calls this *"the single most effective thing on this whole page."* Three decisions:

1. Individually, in pairs, or another way?
2. A **real mechanism** for keeping load even — the guide explicitly rejects intentions
3. What to do when someone finishes early or is stuck a long time

**You already have a division** — the Field Guide's lanes. Putting it on the board makes it real:

| Lane | Owner |
|---|---|
| A · File-and-fix | Natalie finds → **Jason / Mitra** fix → Aisling reviews |
| B · Test untested packages | Jason (`laser-pointer`), Aisling (`fractional-indexing`) |
| C · Docs | Natalie · **Mitra** |
| D · Goodwill triage | Natalie |
| E · MAYBEs, one comment each | Jason (#8586, #6363) · **Mitra (#8067, #11875)** · Aisling (#2236) · Natalie (#11796) |

**Mechanism, not intention:** the Trello board from Part 6 *is* the mechanism. Card counts per
person are visible at a glance. The guide asks who notices a stale card — answer that explicitly
and put a name on it.

**Box and highlight this one.** The guide says so twice.

---

## Part 4 — Handling Conflict *(zone currently mislabelled)*

Four stickies, and the guide wants **two of them boxed**: how you resolve a technical
disagreement, and how you notice someone going quiet.

- Technical disagreement: vote / defer to whoever has most context on that piece / escalate to a
  mentor?
- Someone carrying more than their share — how do they raise it, and to whom?
- Someone going quiet — **how does the team notice, and who says something?** The guide:
  *"Waiting for that person to bring it up themselves is not a plan."*
- Anything from Part 1 that could realistically cause friction?

**The honest answer to the last one for this team** is the Lane A pairing in Part 1 above. Also
worth naming: Aisling reviews every Lane A fix for a11y, and a11y is a declared gap for both
Mitra and Natalie — so review comments will land asymmetrically. Agree now that an a11y redline
from Aisling isn't criticism, it's the lane working.

---

## Part 5 — Our 3 Commitments

Concrete behaviours, not values. **At least one must make it easy to tell early that someone is
stuck or falling behind.** The guide's own contrast:

> Weak: "We'll communicate well."
> Strong: "We'll post a one-line status update in our Slack channel by 9pm on non-class nights."

Three candidates grounded in how this team actually has to operate — **edit these, don't adopt
them unread:**

1. *"Every issue we touch has a Trello card moved to In Progress before the first line of code,
   and no card sits in In Progress more than 48h without a comment saying why."*
   — the early-warning commitment the guide requires.
2. *"Before starting any fix, we comment on the issue to claim it and post the link in Slack.
   No exceptions — the tracker has 1–8 duplicate PRs on every visible issue."*
   — directly addresses the documented failure mode of this repo.
3. *"Every fix PR ships with one regression test and a before/after capture, written by whoever
   made the fix."*
   — matches what maintainers actually merged in the last 7 weeks.

---

## Part 6 — Trello *(fully specifiable now)*

I can't create this — it needs a Trello account and email invites. Everything else is decided:

**Setup:** one teammate creates a free account → new **Workspace** named for the team (the guide
warns: your own Workspace, never a cohort-wide one, or you hit the 10-collaborator cap) → one
board in it → invite all four by email → four lists in this order:

`To Do` → `In Progress` → `In Review` → `Merged/Complete`

**Card discipline:** title = exact issue title · GitHub URL in the description or as a link
attachment · move to In Progress the moment work starts, not retroactively · on PR open, move to
In Review and attach the PR URL as a second link · move to Merged/Complete on merge.

### Starter cards, derived from real work already done

| List | Card | Links |
|---|---|---|
| **In Review**¹ | `fix(editor): re-apply theme-dependent tool cursors on theme change` | issue #11849 |
| To Do | `test(laser-pointer): add unit tests for math and simplify` | Lane B — Jason |
| To Do | `test(fractional-indexing): cover generateKeyBetween / n-keys / validation` | Lane B — Aisling |
| To Do | `docs: remove obsolete CodeSandbox workflow` | issue #11761 — Lane C |
| To Do | `docs: document setViewport, replacing removed scrollToContent` | gated on a #11554 reply |
| To Do | `Comment on #8067 (stroke-width shortcuts)` | Lane E — Mitra |
| To Do | `Comment on #11875 (copy cursor while Alt-duplicating)` | Lane E — Mitra |
| To Do | `Repro: collab room opens on near-empty viewport` | my finding R-1, #11554 area |

¹ The #11849 fix is written, tested (33/33 + negative control), and typecheck-clean locally, but
**no PR exists** — nothing has been pushed. Card belongs in To Do until you actually open it.

---

## What I'd want a mentor to see in two seconds

Your three commitments, boxed, in the Commitments zone — and the work-division answer, boxed, in
Dividing Work. Those are the two the guide singles out. Right now both zones are empty.
