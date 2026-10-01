# ACAE — Frontend Blueprint
### Technical Blueprint · I2EDC Curiosity Projects 2026–27

**Team:** Areebuddin Phundreimayum, Neil Adhikari, Daksh Tomar
**Faculty Mentor:** Nalin Kumar Sharma

This is the single source of truth for the ACAE **frontend**: what the UI
does, how it is built, what is done vs. still planned, and the rules for
building it. It covers `frontend/` only.

**Scope rule.** The frontend and the backend are separate projects with
separate blueprints. **The only thing that binds them is `contract/`**
(`CONTRACT.md`, generated `openapi.json`, generated `examples/`). Everything
the frontend knows about the backend is in those files. This document never
describes backend internals, and if something here seems to need a backend
detail to make sense, that is a gap in the contract: record it (§6, rule 7) and
stop.

**Standing rule:** once a phase's definition-of-done is fully checked, its
lettered sub-phase instructions get collapsed into a short recap row (see the
recap in §9.2 for the pattern).

---

## 1. What the Frontend Is For

ACAE tells a student not just whether an answer was right, but **why their
reasoning failed, and in what recurring shape**, using nine named error
categories. The frontend lets a student take a short quiz on a topic and then
read their results. All diagnosis happens on the other side of the contract;
the frontend only displays what the contract carries.

| Surface | What it shows | Contract endpoints it uses |
|---|---|---|
| Quiz screen | student/topic picker, one question card at a time, options, correct/incorrect feedback after submitting | `GET /topics`, `GET /quiz/next`, `POST /quiz/answer` |
| Results screen | fault-map bar chart by category, dominant-weak-point call-out, remedial note, next-question queue | `GET /results`, `GET /taxonomy` |
| Sample (demo) mode | the same results screen, fed by demo data so it works without real attempts | `POST /sample/generate`, `GET /results/sample` |
| Shell | header (icon + wordmark), light/dark theme toggle, tab title/meta | none |

Display rules that come from the contract (`CONTRACT.md` §1, §3, §9):
- Judgment calls arrive as enumerated **flag codes**, never prose. The frontend
  owns all display text and treats unknown flag codes as ignorable.
- Failures are rendered from `error.code`, never from `message`. Unknown error
  codes render as a generic failure.
- Category names and the list of categories come from `GET /taxonomy`; nothing
  about them is hardcoded.
- `time_taken_sec`, confidences and proportions follow the contract's
  numeric conventions; the frontend never recomputes them.

## 2. The Seam: Contract Only

Inputs a frontend session may rely on, and nothing else:
- `contract/openapi.json` (generated; never hand-edited)
- `contract/CONTRACT.md` (the written standard; where the two disagree the
  generated file wins)
- `contract/examples/` (real payloads, for understanding shapes)

Output: files inside `frontend/`, and only there.

Everything else about the backend (its language, layers, storage, how it is
started) is out of scope for this document and for any frontend session.

## 3. Ownership and Write Scope

| Path | Owner | Others may |
|---|---|---|
| `frontend/` (including `lib/generated/` and `CONTRACT_REQUESTS.md`) | Google AI Studio | read |
| `contract/` | the backend team | frontend sessions read only |

A frontend session **only ever writes inside `frontend/`**. Sessions should be
given `contract/` plus the affected `frontend/` files as context, and no
backend source. The team runs a seam audit around every hand-off (snapshot
before, check after; see `CONTRACT.md` §10) and treats any change outside
`frontend/` as a failed hand-off regardless of how the UI looks.

## 4. Tech Stack

- **Framework:** Next.js (`app/` directory), TypeScript, calling the contract
  over HTTP with `fetch()`.
- **Styling:** plain CSS (CSS Modules + CSS custom properties). No UI component
  library, so the full range of modern CSS (grid, custom properties,
  transitions, `prefers-color-scheme`, container queries where useful) is
  exercised rather than delegated to a framework.
- **Fonts:** `next/font` (self-hosted); no system-default stack.
- **Types:** generated from `contract/openapi.json` by `npm run gen:types`
  into `frontend/lib/generated/`; never hand-edited.
- **Theming:** light and dark token sets over one custom-property contract,
  switched via `data-theme`.
- **Configuration:** `NEXT_PUBLIC_API_BASE_URL` (base URL of whatever serves the
  contract).

## 5. Directory Map

```
frontend/
├── app/                     # pages + components
│   ├── page.tsx             #   quiz screen
│   ├── results/page.tsx     #   results screen
│   ├── components/          #   Header, ThemeToggle, ...
│   ├── globals.css          #   token sets (light/dark)
│   └── layout.tsx, icon.svg
├── lib/
│   ├── api/                 #   the ONLY place that calls fetch()
│   └── generated/           #   types generated from ../contract/openapi.json
├── CONTRACT_REQUESTS.md     # how the frontend asks for contract changes
└── package.json, tsconfig.json, next.config.ts, ...
```

(`lib/api/` and `lib/generated/` arrive with Phase 6.6J; until then the single
`lib/api.ts` plays that role.)

## 6. Conventions

1. **`frontend/lib/api/` is the only code that calls `fetch()`.** Pages and
   components import typed functions (`getNextQuestion()`, `submitAnswer()`);
   they never build URLs.
2. **Types come from the contract.** `npm run gen:types` reads
   `../contract/openapi.json`; generated files are not hand-edited.
3. **The envelope is unwrapped in one place** and failures throw a typed
   `ApiError` carrying `code`, `field` and `request_id`. Components render from
   `code`.
4. **Contract version check.** The client reads the `X-ACAE-Contract` response
   header and, in development, logs a warning when it differs from the hash baked
   into the generated types.
5. **No mock data in `frontend/`.** For development use the contract's sample
   mode (`POST /sample/generate`, `GET /results/sample`) or the real payloads in
   `contract/examples/`.
6. **Answer submissions carry a client-generated `attempt_id`** (UUIDv4), so a
   retry is safe.
7. **Needs something the contract lacks?** Append it to
   `frontend/CONTRACT_REQUESTS.md` and stop. Do not invent an endpoint, do not
   guess a field, do not look at backend source to find out.
8. **Visual work never touches data logic.** Styling phases change no fetching,
   state handling or contract usage.
9. **Accessibility is a hard constraint, not polish:** WCAG AA contrast in both
   themes (verified with a tool), visible focus on every interactive element,
   `aria-live` on answer feedback.

---

## 7. Build Phases — status

| Phase | Scope | Status |
|---|---|---|
| **6** | Next.js frontend: quiz + results screens, sample-mode toggle, calling the contract over `fetch()` | Done |
| **6.5** | Visual + theming redesign — modern typography, high-contrast UI, light/dark theme toggle | 6E–6G done; **6H (fix pass: typography, contrast, dropdown theming) in progress** |
| **6.6J** | Typed client generated from the contract (`lib/api/`, `gen:types`, envelope, `ApiError`) | **Planned — starts when the backend has published `contract/openapi.json` and its audit passes** |
| **8** | Kiosk/tablet UI (large touch targets, focus mode, offline-capable shell) | Not started — roadmap only |

---

## 8. Hand-off Self-Sufficiency Standard

This project is built across many separate sessions, and not necessarily
the same AI assistant each time — one lettered sub-phase might be done in
one chat, the next in a fresh session days later, possibly with a
different model entirely. None of those sessions share memory with any
other, and none of them have read this blueprint beyond whatever gets
attached to them. **The hand-off unit — one lettered sub-phase's write-up
in Section 9 — is the only context a fresh session will ever have.** If
that unit isn't self-sufficient, the hand-off has failed, regardless of
how correct the underlying plan is.

### 8.1 — What "self-sufficient" means, concretely

A sub-phase write-up is self-sufficient only if all of the following hold:

1. **No dangling references.** Never "the above," "as discussed," "per
   our conversation" — a fresh session has no "above," no prior
   discussion. If a decision was made in an earlier sub-phase, either
   inline the decision itself (what was decided, in one sentence) or
   attach the file that already records it.
2. **Open decisions are resolved or explicitly delegated, never just
   gestured at.** If Design direction says "settle which sources are
   in scope," the Prompt must either state the resolved answer or
   explicitly instruct the fresh session to decide AND record one.
3. **Files to attach is the complete, exact set the work needs** —
   nothing "obviously" available from a prior turn.
4. **The Prompt is model-agnostic.** It must read as a complete,
   stand-alone first message: no reliance on a specific AI's prior
   behavior or memory of earlier exchanges in this project.
5. **The After-response check is mechanically verifiable by the fresh
   session itself** — a command to run, a specific observable behavior —
   never something requiring the original author's judgment.

### 8.2 — Required format (standing template, applies to every phase)

Every lettered sub-phase, for every phase, must be written in exactly
this shape and order:

````
#### <ID> — <short title>

**Design direction:** <context, rationale, constraints, open decisions
this sub-phase must settle — tell a fresh session WHY it's doing this
and what to watch for, not just what file to touch. Any decision made
in an earlier sub-phase that this one depends on gets restated here in
one sentence, not referenced by "as decided above.">

**Files to attach:** `path/one.py`, `path/two.py`

**Prompt:**
> <a self-contained, imperative instruction a fresh chat with only the
> attached files can act on immediately — never "see above.">

**After-response check:** <one concrete, mechanically checkable thing
that proves the sub-phase actually worked>

---
````

The `**Prompt:**` block is not a copy of `**Design direction:**` — Design
direction is background for whoever is deciding what this sub-phase
should do; Prompt is the tightened, self-contained instruction for
whoever is about to actually do it.

**Sub-phase sizing rule (standing rule, applies to every phase from now
on):** each lettered sub-phase must cover exactly one file (or one
clearly-bounded addition to one file) and one concern, sized to fit
inside a single chat session's context without running out of tokens
mid-response. If a sub-phase would require holding more than one new
concept in play at once — e.g. "write the scraper AND parse the PDF AND
tag the result" — split it into further lettered sub-phases *before*
starting, not after hitting a context wall. This is why Phase 6.5 below is split into lettered parts: tokens, quiz screen,
results screen and the fix pass are separate concerns, each with its own
failure mode, so each gets its own step.

---

## 9. Step-by-Step Instructions — Current Phase

Completed phases are condensed to short recaps — the working code and git
history are the source of truth for exactly how they were built.

### 9.1 — Recap: Phase 6, Next.js frontend (done)

The frontend is a standalone Next.js app in `frontend/`:
```
cd frontend && npm install && npm run dev    # http://localhost:3000
```
It needs a server that implements the contract at `NEXT_PUBLIC_API_BASE_URL`;
how that server is started is not this document's concern. It replicates the
quiz and results flows (picker, question card, feedback; fault map, dominant
weak point, remedial note, next-question queue) via `fetch()`, plus the
sample/demo-mode toggle. The question payload deliberately carries no answer
information; correctness appears only in the response to an answer submission.

---

### 9.2 — Phase 6.5: Frontend Theming & Visual Redesign (in progress)

**Goal:** Phase 6 made the Next.js frontend functionally complete but
visually plain. Phase 6.5 makes it look intentional: **modern typography,
strong/accessible color contrast, and a proper light/dark theme system
with a user-facing toggle** — plus the branding cleanup (icon, full "ACAE"
title). **Visual-only — no diagnostic logic, API contract, or
state-management behavior changes anywhere in this phase.**

**Design direction to give every sub-phase below, verbatim, so the result
stays consistent across separate AI-assistant sessions:**
> Minimalist, education-focused visual design. A calm, focused
> reading/quiz-taking experience — generous white space, one clear focal
> point per screen, restrained color used only for meaning
> (correct/incorrect/category color-coding), not decoration.
> **Typography:** pick one modern, highly-legible sans (e.g. a
> humanist/grotesque sans like Inter, Manrope, or similar — self-hosted or
> via `next/font`, not a system-default stack) for UI/body text, optionally
> paired with a second display font for headings only if it stays legible
> at small sizes; let type hierarchy do most of the visual work instead of
> borders/shadows/gradients.
> **Contrast:** every text/background pairing must meet WCAG AA (4.5:1 for
> body text, 3:1 for large text/UI components) in *both* themes — treat
> this as a hard constraint, not a nice-to-have, and verify it, don't just
> assert it.
> **Theming:** implement light and dark themes as two token sets sharing
> one CSS custom-property contract (e.g. `--color-bg`, `--color-text`,
> `--color-accent`, `--color-correct`, `--color-incorrect`, ...), default
> to the user's OS preference (`prefers-color-scheme`) on first load, and
> expose a persistent manual toggle (localStorage-backed) that overrides
> it — semantic colors (correct/incorrect/category coding) must stay
> legible and keep the same *meaning* in both themes, not just invert.
> Use plain CSS (CSS Modules + CSS custom properties) — no UI component
> library — so the full range of modern CSS (grid, custom properties,
> transitions, `prefers-color-scheme`, container queries where useful) is
> actually exercised rather than delegated to a framework's defaults.

**Who this section is for:** hand each lettered part below to Google AI
Studio in its own fresh conversation, attach exactly the files listed,
paste the exact prompt given. **Google AI Studio only ever writes inside
`frontend/`.** The team runs the seam audit (`--snapshot` immediately before
every handoff, `--check` immediately after; see `CONTRACT.md` §10), including
for 6H below and any future regeneration.

#### 6E–6G — Recap: tokens/shell, quiz redesign, results redesign (done)

| Part | Built | Files |
|---|---|---|
| 6E | Light/dark CSS custom-property token sets switched via `data-theme`, a modern sans via `next/font`, `ThemeToggle` (reads `prefers-color-scheme`, persists to `localStorage`, no flash-of-wrong-theme), `app/icon.svg`, shared `Header` (icon + wordmark + toggle), tab title/meta | `globals.css`, `layout.tsx`, `components/Header.tsx(+.module.css)`, `components/ThemeToggle.tsx(+.module.css)`, `icon.svg` |
| 6F | Quiz-taking screen (student/topic picker, question card, options, correct/incorrect feedback) restyled on the 6E tokens; same `fetch()`/state logic | `app/page.tsx`, `app/page.module.css` |
| 6G | Results screen (fault-map bar chart, dominant-weak-point call-out, remedial note, next-question queue, sample-result toggle) restyled on the 6E tokens; same `/api/results` data and status handling | `app/results/page.tsx`, `app/results/results.module.css` |

**Issues found in review, not yet fixed** (scope of 6H below):
- Typography doesn't read as "modern, elegant" yet.
- Several text/background and UI pairings fall short of WCAG AA in
  practice — asserted in 6E/6F/6G, not independently verified.
- Native `<select>` dropdowns don't pick up theme tokens at all — in dark
  theme they show as a light-background dropdown with barely-legible text.

---

#### 6H — Fix pass: typography, contrast, and dropdown theming (current target)

**Design direction:** a targeted fix pass on top of the 6E–6G redesign —
not a new visual style, a correction of the three issues above — plus the
same responsive/motion/accessibility sweep originally scoped for this
step. The 6E–6G redesign is functionally done but has three known
problems: (1) typography doesn't read as modern/elegant; (2) several
text/background pairings fail WCAG AA; (3) native `<select>` elements
ignore the theme system entirely, showing unreadable light-on-light or
dark-on-dark chrome depending on OS/browser default. Fix all three, no new
features, no data-logic changes.

**Files to attach:** the full `frontend/app/` directory as it stands
after 6G (tokens, header, theme toggle, quiz page, results page).

**Prompt:**
> This is Phase 6.5 (part 6H, fix pass) of ACAE. [paste the design
> direction paragraph from §9.2 above]. The 6E–6G redesign (attached) is
> functionally done but has three known problems to fix, no new
> features: (1) **Typography** — the current font/type-scale doesn't read
> as modern/elegant; reconsider the font choice (or its weight, tracking,
> and line-height) and the heading/body type scale until it does, keeping
> it via `next/font`. (2) **Contrast** — audit every text/background and
> semantic-color pairing in *both* themes against WCAG AA (4.5:1 body,
> 3:1 large text/UI) using an actual contrast-checking tool, not
> eyeballing, and fix every pairing that fails — list which token pairs
> you changed and their new ratios. (3) **Dropdowns** — every `<select>`
> element (the topic/student picker, and any other native select) must
> visually follow the theme tokens in both light and dark mode:
> background, text, border, and the options-list popup itself, not just
> the closed control. If native `<select>` styling can't be made to fully
> respect `data-theme` cross-browser, replace it with a custom-styled
> listbox/combobox built from the same tokens instead of leaving it
> unthemed — same options, same `onChange` behavior, no data logic
> change. Also do the standard polish pass: responsive layout down to
> ~375px width, consistent restrained transitions on one shared
> easing/duration, visible focus ring on every interactive element in
> both themes, and `aria-live` on the answer-feedback region. Don't touch
> data-fetching or state logic anywhere.

**After-response check:**
0. Before pasting the prompt to Google AI Studio, run
   `python scripts/audit_frontend_integration.py --snapshot`.
1. Copy in, `npm run dev`.
2. Open the topic/student picker dropdown in **dark theme specifically**
   — confirm it no longer shows as a light box with unreadable text.
3. Spot-check contrast with an actual tool on the pairings the response
   says it fixed — confirm the ratios hold, don't just trust the claim.
4. Resize down to ~375px in both themes — confirm nothing overflows.
5. Tab through both pages keyboard-only in both themes — confirm a
   visible focus ring everywhere.
6. Toggle theme repeatedly across both pages — no flash, no element stuck
   in the wrong theme's color, dropdown included.
7. Run `python scripts/audit_frontend_integration.py --check` — it must
   report zero non-frontend changes; if it reports any, treat that as a
   failed handoff regardless of how the UI looks, and don't merge until
   resolved.
8. This is the last step of Phase 6.5 — update §7's status table row for
   6.5 to "Done" once all boxes below are checked.

**Definition of done for all of Phase 6.5 (6E–6H):**
- [ ] Browser tab shows the icon and full "ACAE — Adaptive Cognitive
      Assessment Engine" title
- [ ] Typography genuinely reads as modern/elegant, not just "a
      non-default font is loaded"
- [ ] Light/dark themes exist, default to OS preference, switchable via a
      persistent toggle, no flash-of-wrong-theme
- [ ] Every text/background and semantic-color pairing meets WCAG AA in
      both themes, **verified with a contrast tool, not eyeballed**
- [ ] Every `<select>`/dropdown follows the theme tokens in both modes,
      including the open options list
- [ ] Quiz and results pages share one visual language
- [ ] No diagnostic logic, API calls, or state handling changed anywhere
      in 6E–6H — visual-only
- [ ] Layout holds at desktop and ~375px mobile width
- [ ] Every interactive element has a visible keyboard focus state in
      both themes
- [ ] `scripts/audit_frontend_integration.py --check` reports zero
      non-frontend changes after the Google AI Studio handoff

---

### 9.3 — Phase 6.6J: Typed client generated from the contract

**Precondition:** `contract/openapi.json`, `contract/CONTRACT.md` and
`contract/examples/` exist and are current. If they do not, stop; this is not
frontend work to invent.

#### 6.6J — Frontend typed client (Google AI Studio)

**Design direction:** the only frontend phase. The frontend gets one place
that talks to the backend, typed from the generated contract. Visual design,
routing and state behavior are unchanged. Run
`python scripts/audit_frontend_integration.py --snapshot` immediately before
the handoff and `--check` immediately after.

**Files to attach:** `contract/openapi.json`, `contract/CONTRACT.md`,
`contract/examples/` (all files), `frontend/lib/api.ts`,
`frontend/app/page.tsx`, `frontend/app/results/page.tsx`,
`frontend/package.json`.

**Prompt:**
> This is Phase 6.6J of ACAE. You may only write inside `frontend/`. Attached
> are the backend contract (OpenAPI plus a written standard plus real example
> payloads) and the current pages. Add an `npm run gen:types` script that
> generates TypeScript types from `../contract/openapi.json` into
> `frontend/lib/generated/`. Replace `frontend/lib/api.ts` with a
> `frontend/lib/api/` module: one typed function per endpoint in the contract,
> targeting `/api/v1`, reading the base URL from `NEXT_PUBLIC_API_BASE_URL`,
> unwrapping the `{ok, data, meta}` envelope in a single place, throwing a typed
> `ApiError` with `code`, `field` and `request_id`, generating a UUID
> `attempt_id` for each answer submission, and logging a development warning
> when the `X-ACAE-Contract` response header differs from the hash in the
> generated types. Update the two pages to call these functions and render
> from `error.code`. Do not change any styling, layout, or component
> structure. Do not embed mock data; if you need an endpoint or field the
> contract lacks, add it to `frontend/CONTRACT_REQUESTS.md` and stop.

**After-response check:** `cd frontend && npm run gen:types && npm run build`
succeed; a full quiz plus results flow works against the running backend; and
`python scripts/audit_frontend_integration.py --check` exits 0.

---

**Definition of done for Phase 6.6J:**
- [ ] `frontend/` talks to the contract only through `frontend/lib/api/`
- [ ] `npm run gen:types` and `npm run build` succeed
- [ ] No mock question/result data anywhere in `frontend/`
- [ ] Styling, layout and component structure unchanged
- [ ] The seam audit's `--check` passes after the hand-off

---

### 9.4 — Phase 8: Kiosk / Tablet UI (roadmap only — layman's plan)

**Goal:** make the frontend usable standalone on a low-spec tablet at a physical
kiosk: large touch targets, a dim/focus mode, minimal chrome, and no dependency
on an internet connection once installed.

**Not worth starting until Phase 6.5 is done and demoed.** It also depends on
hardware decisions (which tablet? which OS?) not made yet, so it's
roadmap-level, not exact copy-paste prompts.

- **8A — Prove the UI is offline-clean.** Confirm the only network requests the
  app makes are to the contract's base URL (no CDN fonts, no analytics, no
  remote images). Verify with the network panel and with Wi-Fi off against a
  locally running contract server.
- **8B — Package it.** `npm install` plus a production build; investigate a
  static Next.js export as its own session if per-tablet setup isn't practical.
- **8C — Kiosk-mode the browser.** OS-level config on the actual tablet —
  full-screen, no address bar, auto-launch on boot. Steps depend entirely on the
  tablet OS chosen.
- **8D — Pilot on one real device** before duplicating — watch for touch
  targets too small and text too small at arm's length, in both themes.

---

## 10. Quick Reference

```
cd frontend
npm install
npm run gen:types      # regenerate lib/generated/ after contract/ changes
npm run dev            # http://localhost:3000
npm run build
```
Set `NEXT_PUBLIC_API_BASE_URL` to the base URL of a server that implements the
contract. A root-level convenience script may launch the frontend alongside
other things; this blueprint does not depend on it.

## 11. Working Inside Claude Projects / Google AI Studio

Each lettered sub-phase is handed to Google AI Studio in its own fresh
conversation: attach exactly the files listed under **Files to attach**, paste
the exact **Prompt**, and run the **After-response check**. Context given to a
session is `contract/` plus the listed `frontend/` files, and nothing from the
backend. Human-judgment checks (typography, contrast, dropdown theming) are
signed off by a person after working through the checklist; a script never
marks them done.
