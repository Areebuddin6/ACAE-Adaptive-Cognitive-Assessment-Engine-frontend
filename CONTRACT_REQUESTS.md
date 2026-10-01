# ACAE Frontend → Contract Requests

**Location:** `frontend/CONTRACT_REQUESTS.md` (owned by the frontend; the backend team only reads it).

This file is the frontend's only channel for asking the contract to change. It exists because
the frontend may rely on `contract/` and nothing else: it does not read backend source, does not
guess a field, and does not invent an endpoint. When the contract lacks something, the session
records it here and stops.

## How this file works

1. **Append only.** Add new requests at the bottom with the next `CR-nnn` id. Never delete or
   renumber. A request that turns out to be unnecessary is marked `withdrawn`, with a reason.
2. **A human relays it.** The team copies open requests into a backend session. The backend
   decides, edits `backend/api/schemas.py`, regenerates `contract/`, and runs the seam audit
   (`CONTRACT.md` §9). The frontend never learns the answer by reading backend code.
3. **Proposals are suggestions.** Each "Proposed shape" is the smallest shape the frontend
   could use. The backend may choose a different one. Once `contract/openapi.json` has the
   result, `openapi.json` wins over anything written here.
4. **Status values:** `open` → `relayed` → `resolved (contract <hash>)` or `withdrawn`.
5. **Additive only.** Every proposal is designed to be legal inside `v1` (`CONTRACT.md` §9):
   new optional fields, new endpoints, or clarifications. None needs a `v2`.

### First step for any session that opens this file

Compare each `open` request below against the current `contract/openapi.json` and
`contract/examples/`. If the contract already satisfies it, mark it
`withdrawn (already in contract)` and move on. These requests were written from the written
standard (`CONTRACT.md`) before the generated file was available to check against.

## Priority legend

- **P1:** blocks Phase 6.6J (typed client) or the quiz → results flow.
- **P2:** needed for correct UI states; the UI can ship a degraded version without it.
- **P3:** nice to have.

## Index

| ID | Title | Priority | Status |
|---|---|---|---|
| CR-001 | How does the picker get the list of students? | P1 | open |
| CR-002 | Exact algorithm for the contract hash | P1 | open |
| CR-003 | CORS must expose and allow the contract header | P1 | open |
| CR-004 | Every failure must use the error envelope | P1 | open |
| CR-005 | Define `time_taken_sec` on `POST /quiz/answer` | P1 | open |
| CR-006 | Remedial note: authored text or enumerated data? | P2 | open |
| CR-007 | `GET /quiz/next` when the bank is exhausted | P2 | open |
| CR-008 | `GET /results` with zero or few attempts | P2 | open |
| CR-009 | Sample mode: what links generate to results? | P2 | open |
| CR-010 | Taxonomy and topic display fields | P2 | open |
| CR-011 | `contract/examples/` must cover every state | P2 | open |
| CR-012 | Question count on each topic | P3 | open |

---

## CR-001: How does the picker get the list of students?

- **Priority:** P1 · **Status:** open · **Blocks:** quiz screen picker under 6.6J
- **Need:** The quiz screen has a student/topic picker (Frontend Blueprint §1). The contract has
  `GET /topics` for topics but no way to list students. `student_id` is optional on requests
  and the server resolves a default.
- **Why:** Without a list, the student half of the picker cannot be a dropdown. The frontend
  will not hardcode ids and will not use mock data.
- **Proposed shape:** `GET /students` returns `{ "students": [ { "student_id": "…", "label": "…" } ] }`,
  as a query with the standard envelope.
- **Alternative:** If the product decision is "one default student, no picker", say so in
  `CONTRACT.md`. The frontend then removes the student half of the picker and sends no
  `student_id`.
- **Acceptance:** `openapi.json` contains either a students endpoint or an explicit statement
  that the student picker does not exist. The frontend can build the picker from the contract
  alone.

## CR-002: Exact algorithm for the contract hash

- **Priority:** P1 · **Status:** open · **Blocks:** the dev warning required by 6.6J
- **Need:** `CONTRACT.md` §8.4 says the backend stamps `X-ACAE-Contract` with "the short hash of
  `openapi.json`", and that the client compares it with "the hash baked into the generated
  types". Neither the hash function, the length, nor how it gets into the generated types is
  specified. A hash cannot live inside the file it hashes.
- **Why:** If the frontend picks a different algorithm from the backend, the warning fires
  always or never, silently.
- **Proposed shape:** State one rule in `CONTRACT.md`, for example "first 12 lowercase hex
  characters of SHA-256 over the exact bytes of `contract/openapi.json`". The frontend's
  `gen:types` script computes it from the file it reads, writes it into `lib/generated/`, and
  the header and `GET /health.contract_hash` use the identical value.
- **Acceptance:** Run the stated rule over `contract/openapi.json`; the result equals
  `X-ACAE-Contract` on a live response and `contract_hash` from `/health`.

## CR-003: CORS must expose and allow the contract header

- **Priority:** P1 · **Status:** open · **Blocks:** any browser call from the dev server to the API
- **Need:** The frontend runs on a different origin from the API (`NEXT_PUBLIC_API_BASE_URL`).
  Browsers hide non-safelisted response headers from JavaScript unless the server lists them
  in `Access-Control-Expose-Headers`. They also preflight `POST` requests with a JSON body and
  custom request headers.
- **Why:** Without this, `response.headers.get("X-ACAE-Contract")` returns `null` in the
  browser and the version check never works, with no visible error.
- **Proposed shape:** Document in `CONTRACT.md` §2 that every response, including errors and
  `OPTIONS` preflights, carries CORS headers for the origins in `ACAE_ALLOWED_ORIGINS`, that
  `Access-Control-Expose-Headers` lists `X-ACAE-Contract` (and `X-Request-ID` if one is added),
  and that `Access-Control-Allow-Headers` accepts `Content-Type` and `X-ACAE-Contract`.
- **Acceptance:** From a page on `http://localhost:3000`, a `fetch()` to `/api/v1/health`
  can read `X-ACAE-Contract`, and a `POST /api/v1/quiz/answer` succeeds after its preflight.

## CR-004: Every failure must use the error envelope

- **Priority:** P1 · **Status:** open · **Blocks:** rendering errors from `error.code`
- **Need:** `CONTRACT.md` §3 says every response, success or failure, has the envelope. The
  client unwraps it in one place and branches on `error.code`. Web frameworks often return
  their own bodies for unknown routes, wrong methods, malformed JSON and validation failures.
- **Why:** One un-enveloped failure body breaks the single unwrapping point.
- **Proposed shape:** Add a sentence to §4 stating that unknown paths, unsupported methods,
  unparseable JSON and schema-validation failures all return `ok: false` with a code from the
  closed set. State the mapping for the first three (for example `not_found` for unknown
  paths; `validation_error` for the other two; say if a 405 needs a new code). Also state how
  `field` is written for nested request errors (for example a dotted path).
- **Acceptance:** `contract/examples/` has an example for each of the above, and each parses
  as the error envelope.

## CR-005: Define `time_taken_sec` on `POST /quiz/answer`

- **Priority:** P1 · **Status:** open · **Blocks:** the answer-submission function in `lib/api/`
- **Need:** `CONTRACT.md` §2 says durations are integer seconds with a `_sec` suffix, and the
  Frontend Blueprint §1 mentions `time_taken_sec`. The written standard does not say who
  measures it, whether it is required, or what its bounds are.
- **Why:** Timing evidence changes the diagnosis (`timing_signal_only`). The frontend must
  measure the same thing the server expects.
- **Proposed shape:** State in `CONTRACT.md` and in the schema description: measured by the
  client from the moment the question is displayed to the moment of submission, integer, `>= 0`,
  rounded down (or whichever rule the backend wants). Say whether it is required or optional,
  and what the server does when it is absent or implausibly large (rejects with
  `validation_error`, or ignores it).
- **Acceptance:** `AnswerRequest` in `openapi.json` marks `time_taken_sec` as required or
  optional, with a `minimum` and a description of who measures it.

## CR-006: Remedial note: authored text or enumerated data?

- **Priority:** P2 · **Status:** open · **Blocks:** the remedial-note block on the results screen
- **Need:** `CONTRACT.md` §1.4 says the API carries "data and flags, never prose meant for the
  UI", and that display text lives in the frontend. §5 also says `GET /results` returns a
  "remedial note". These two statements pull in opposite directions.
- **Why:** The frontend cannot tell whether it should render a string the API supplies or map a
  code to its own text. Guessing wrongly means either a blank block or text that duplicates
  and then drifts from the backend's remedial mapping.
- **Proposed shape (preferred):** `remedial: { "category_id": CategoryId, "note": string } | null`,
  with the contract stating explicitly that `note` is **authored curriculum content** (a
  remedial explanation, not a judgment call) and is the one allowed exception to §1.4.
  **Acceptable alternative:** `remedial: { "category_id": CategoryId, "note_code": NoteCode } | null`
  with a closed enum, in which case the frontend owns the text.
- **Acceptance:** `ResultsData` in `openapi.json` has a typed `remedial` field and §1.4
  mentions how it is treated.

## CR-007: `GET /quiz/next` when the bank is exhausted

- **Priority:** P2 · **Status:** open · **Blocks:** the question card's end states
- **Need:** `CONTRACT.md` §4 and §7 say running out of questions is a `200` carrying
  `bank_exhausted_repeat` or `remedial_note_only`. They do not say what `data` looks like in
  those cases, especially when there is no question to show.
- **Why:** The quiz screen needs to know whether to render a question, a "you have seen these
  before" state, or the remedial note instead of a question.
- **Proposed shape:** `QuizNextData = { student_id, topic_id, question: QuestionPublic | null,
  remedial: Remedial | null, flags: FlagCode[] }`. With `bank_exhausted_repeat`, `question` is
  the repeated question. With `remedial_note_only`, `question` is `null` and `remedial` is set.
  Otherwise `flags` is empty and `remedial` is `null`.
- **Acceptance:** The shape above (or the backend's equivalent) is in `openapi.json`, with
  examples for the normal, repeat and note-only cases.

## CR-008: `GET /results` with zero or few attempts

- **Priority:** P2 · **Status:** open · **Blocks:** the results screen empty and provisional states
- **Need:** The standard defines `provisional_profile` (fewer than 8 attempts) and
  `no_systematic_pattern`, but not the shape of the data underneath them: whether the
  per-category list is complete, whether the dominant category is `null`, and whether the
  number of attempts is reported.
- **Why:** The fault-map chart should always render all categories in a stable order, and the
  call-out should know whether there is a dominant category at all. The frontend must not
  recompute either.
- **Proposed shape:** The per-category list always contains every category from
  `GET /taxonomy`, in taxonomy order, with explicit zeros where there is no evidence.
  `dominant_category` is `CategoryId | null` (null when `no_systematic_pattern` is present).
  Include `attempt_count: integer`. Flags sit on the results object.
- **Acceptance:** Examples exist for: no attempts at all, a provisional profile, a normal
  profile, and a profile with `no_systematic_pattern`.

## CR-009: Sample mode: what links generate to results?

- **Priority:** P2 · **Status:** open · **Blocks:** the demo-mode toggle
- **Need:** `POST /sample/generate` creates demo data and `GET /results/sample` reads it. The
  contract does not say what the generate call returns, whether it takes parameters, whether
  calling it twice duplicates data, or what `/results/sample` returns before anything has been
  generated.
- **Why:** The toggle must show a loading state, a ready state or an "empty demo" state
  without guessing. Under the safe-to-re-run rule it should be repeatable.
- **Proposed shape:** `POST /sample/generate` takes no required body and returns the resolved
  `student_id`, `topic_id` and `attempt_count` of the demo data. It is idempotent: repeating
  it replaces the demo data and never grows it. `GET /results/sample` before generation
  returns `200` with the same shape as an empty `/results` (see CR-008), not an error.
- **Acceptance:** Both response schemas are in `openapi.json`, and the "before generation"
  behaviour is stated in `CONTRACT.md` §5.

## CR-010: Taxonomy and topic display fields

- **Priority:** P2 · **Status:** open · **Blocks:** labels in the picker, fault map and call-out
- **Need:** The frontend must take category names and the category list from `GET /taxonomy`,
  and topics from `GET /topics`. `CONTRACT.md` §7 says the field sets come from existing
  payloads and are not defined in the document.
- **Why:** The frontend has no other source for a human-readable label. It will not derive
  labels by reformatting ids like `time_pressure_collapse`.
- **Proposed shape:** Each taxonomy entry has `category_id: CategoryId`, `name: string` (a
  short display name) and `description: string` (one line, authored). The list order is
  stable and is the order the UI uses. Each topic has `topic_id: string` and `name: string`.
- **Acceptance:** These fields are present and typed in `openapi.json`, and appear in the
  examples for both endpoints.

## CR-011: `contract/examples/` must cover every state

- **Priority:** P2 · **Status:** open · **Blocks:** developing without mock data
- **Need:** Frontend Blueprint rule 5 and `CONTRACT.md` §8.5 forbid mock data in `frontend/`.
  The only offline reference for shapes is `contract/examples/`.
- **Why:** A frontend that can only see the happy path cannot build or check the other states
  without inventing payloads.
- **Proposed shape:** One real payload per endpoint, plus: one example per `FlagCode`, one
  per `ErrorCode`, an answer replay (same `attempt_id` and payload sent twice), and the
  states from CR-007, CR-008 and CR-009. Name files `<endpoint>.<state>.json`.
- **Acceptance:** `contract/examples/` lists a file for each item above, and each validates
  against its response schema.

## CR-012: Question count on each topic

- **Priority:** P3 · **Status:** open · **Blocks:** nothing
- **Need:** The picker would like to avoid offering a topic with no questions.
- **Why:** Without this, the user picks an empty topic and meets the end state from CR-007
  immediately.
- **Proposed shape:** Add an optional `question_count: integer` to each `GET /topics` entry
  (an additive, optional field, so legal in `v1`).
- **Acceptance:** The field appears in `openapi.json` and in the topics example.

---

## Decisions the frontend makes itself (no contract change needed)

Recorded here so no session raises them as requests.

- Unknown `FlagCode` values are ignored; unknown `ErrorCode` values render the generic failure.
- All display text for flags and error codes lives in `frontend/`.
- The `attempt_id` is a UUIDv4 generated in `lib/api/` when an answer is submitted. A retry
  after a network failure reuses the same id.
- Colors for categories belong to the theme tokens in `globals.css`, keyed by `category_id`.
  The contract carries no color or styling information.
