# Fixproof — Project Plan

> **Goal:** Given an accessibility scan result produced by ada-remediation,
> call an LLM to propose a concrete HTML fix, apply it safely to an isolated copy,
> rescan the copy, and prove (or disprove) that the finding disappeared.

This document is a living plan. Each section maps to a Git branch / milestone.
No code is written here — only the intent for each step.

---

## Phase 0 — Foundation (done ✅)

- [x] Cloned upstream, confirmed `npm run check` passes (47 tests, 0 errors)
- [x] Ran CLI scan of demo site, verified 14 findings on 5 pages
- [x] Read all source files; understand the `Finding` type and `remediationPrompt` field
- [x] Added Fixproof copyright line and README section
- [x] Created this plan

---

## Phase 1 — Test pages (`fixproof/tests/fixtures/`)

**Goal:** Have a small set of HTML files with *known, deliberate* accessibility errors
and *known correct fixes* so every later phase has ground truth to check against.

**Plan:**
- Create 5–8 minimal HTML pages, one per rule type that the demo site exercises:
  - `aria-required-parent` (nav links with wrong ARIA role)
  - `image-alt` (missing alt text)
  - `color-contrast` (text with insufficient contrast)
  - `html-has-lang` (missing lang attribute)
  - `button-name` (unlabelled button)
  - `label` / `select-name` (form controls without labels)
- For each fixture, also write the *expected fixed version* — this is what a
  correct LLM response should produce (used in integration tests without hitting a real API)
- Store fixtures under `fixproof/tests/fixtures/broken/` and `fixproof/tests/fixtures/fixed/`

**No model calls. No upstream changes.**

---

## Phase 2 — Fix proposer (`fixproof/src/llm/`)

**Goal:** Given a single `Finding` object from a scan result, produce a proposed
HTML patch by calling an LLM.

**Plan:**
- Build `fixproof/src/llm/proposer.ts`:
  - Input: `Finding` (has `remediationGuidance`, `evidence`, `codeSuggestion`)
  - Assemble a prompt from the finding's existing `remediationPrompt` field
    (already built by `src/guidance.ts`) — no new prompt engineering needed for v1
  - Call the LLM via the provider's official SDK (provider selected at runtime via env var)
  - Parse the response: expect a `before` / `after` HTML snippet
  - Return a typed `ProposedFix { findingFingerprint, before, after, rationale, modelId }`
- API keys come from `.env` only (`OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, etc.)
- A `--dry-run` flag returns a mock fix without calling any paid API
  (uses the `codeSuggestion.after` field that the upstream already computes)

**No upstream files touched. No keys in code.**

---

## Phase 3 — Safe patcher (`fixproof/src/patcher/`)

**Goal:** Apply a proposed fix to an *isolated copy* of the page source,
never touching the original.

**Plan:**
- Build `fixproof/src/patcher/isolate.ts`:
  - Copy the target HTML file to a temp directory (or in-memory string)
  - Apply the `before → after` substitution using the same `replaceNearest()`
    logic that already exists in upstream `src/remediation.ts` (import, don't copy)
  - Write the patched copy; return its local `file://` URL for the verifier
- Constraints:
  - Only one finding is patched at a time (avoids interactions between fixes)
  - The original file is read-only; the patcher writes to a temp path
  - If the `before` text is not found in the source, the patcher returns `PatchResult.NOT_FOUND`
    rather than silently succeeding

---

## Phase 4 — Verifier (`fixproof/src/verifier/`)

**Goal:** Rescan the patched copy and determine whether the target finding is gone.

**Plan:**
- Build `fixproof/src/verifier/verify.ts`:
  - Call `scanUrls([patchedFileUrl], options)` from the upstream scanner
    (import directly — it is already a clean function)
  - Receive a new `ScanResult`
  - Check whether the original `finding.fingerprint` appears in the new result
  - Return a typed `VerificationResult`:
    - `FIXED` — the fingerprint is gone
    - `STILL_PRESENT` — the fingerprint remains
    - `NEW_VIOLATIONS` — the patch introduced a different finding
    - `SCAN_ERROR` — the rescan itself failed

---

## Phase 5 — Benchmark reporter (`fixproof/src/reporter/`)

**Goal:** Run Fixproof over all fixtures and produce a concise table of results.

**Plan:**
- Build `fixproof/src/reporter/benchmark.ts`:
  - For each fixture in `tests/fixtures/broken/`:
    - Scan it → get findings
    - For each finding, call the proposer (dry-run mode)
    - Apply the patch, rescan, classify the result
  - Produce a Markdown table:

    | Fixture         | Rule                  | Patch source    | Result  |
    |-----------------|-----------------------|-----------------|---------|
    | image-alt.html  | image-alt             | codeSuggestion  | FIXED   |
    | nav-aria.html   | aria-required-parent  | LLM (dry-run)   | FIXED   |

  - Write to `fixproof/results/benchmark-<timestamp>.md`
- **No numbers are invented.** The table is generated from actual scan output.

---

## Phase 6 — Tests (`fixproof/tests/`)

**Goal:** Every function in Phases 1–5 has a unit test that runs without a browser or API call.

**Plan:**
- Unit tests (Vitest, same runner as upstream):
  - `proposer.test.ts` — mock the LLM client; assert correct prompt structure and parse
  - `patcher.test.ts` — feed known HTML + before/after; assert correct output
  - `verifier.test.ts` — mock `scanUrls`; assert each `VerificationResult` case
  - `reporter.test.ts` — feed fixture scan results; assert Markdown table format
- Integration test (requires Playwright, opt-in via `FIXPROOF_INTEGRATION=1`):
  - Run full Fixproof loop on `tests/fixtures/broken/image-alt.html` in dry-run mode
  - Assert the benchmark table contains `FIXED` for that fixture
- All tests run with `npm run test` from `fixproof/` without hitting any paid API

---

## Phase 7 — CI (`.github/workflows/fixproof-ci.yml`)

**Goal:** Every pull request on this fork runs the Fixproof unit tests.

**Plan:**
- Add a GitHub Actions workflow `fixproof-ci.yml`:
  - Trigger: push and pull_request to `main`
  - Steps: checkout → `npm install` (root) → `npm install` (fixproof) →
    `npx playwright install chromium` → `npm run check` (root, upstream tests) →
    `npm test` (fixproof unit tests)
  - The integration test is excluded from CI (requires a real API key or large fixture)
- No secrets are stored in the workflow file — the workflow prints a warning
  if `OPENAI_API_KEY` is absent and skips the live-model path

---

## Target directory layout

```
ada-remediation/              <- upstream (never rewritten)
  src/
  tests/
  docs/GUIDE.md
  LICENSE                     <- two copyright lines
  README.md                   <- Fixproof section prepended

  fixproof/                   <- everything Fixproof
    src/
      llm/
        proposer.ts           <- Phase 2
      patcher/
        isolate.ts            <- Phase 3
      verifier/
        verify.ts             <- Phase 4
      reporter/
        benchmark.ts          <- Phase 5
      index.ts                <- Fixproof CLI entry
    tests/
      fixtures/
        broken/               <- Phase 1 test pages
        fixed/                <- Phase 1 expected outputs
      unit/
      integration/
    docs/
      FIXPROOF_PLAN.md        <- this file
    .env.example              <- OPENAI_API_KEY=, ANTHROPIC_API_KEY=
    package.json
    tsconfig.json
```

---

## What I will NOT do

- Rewrite any upstream file (unless a genuine bug is found and upstreamed separately)
- Store API keys anywhere in the repository
- Scan any site I do not own
- Make live LLM calls during unit tests or CI
- Invent numbers in the benchmark table
