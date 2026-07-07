# GapMap — Design Spec

**Date:** 2026-07-07 · **App #17 in Moiz's 30-in-30** · **Status:** approved design, ready for implementation planning

> **READ THIS FIRST (note to the implementing model, e.g. Opus):** This spec is deliberately self-contained. Every prompt, type, formula, and rule you need is written out literally — do not invent alternatives or "improve" the honesty rules. Moiz's global `~/.claude/CLAUDE.md` applies in full (design tier, teaching comments, pre-flight checklist, AI feature rules, subagent delegation). Where this spec and generic best practice disagree, this spec wins. Where this spec is silent, CLAUDE.md wins. Process: invoke `superpowers:writing-plans` from this spec, then execute with `/code-review` after each implementation step, `superpowers:frontend-design` BEFORE any UI code, `superpowers:webapp-testing` before claiming it works.

---

## 1. What GapMap is

Paste a resume + a job description → get a hiring manager's first-pass screen, made transparent to the candidate:

- Every JD requirement judged **match / partial / missing**
- Every judgment tied to **verbatim quoted evidence from the resume** — programmatically verified, fabrication structurally impossible to display
- A **concrete suggestion per gap** (rewrite this bullet / add this project / learn this skill) — never "improve your skills" fluff
- An **honest, explainable readiness score** computed in code (never by the LLM), with the formula shown
- A **shareable score-card image** (no resume text on it) for the LinkedIn viral loop

This is also Moiz's FDE portfolio piece: the pipeline (extract → judge → verify → score) ships with an eval, and the production model is chosen by measurement (Haiku vs Sonnet on the judge stage), not vibes.

**Ship as ONE build — full vision, no phases.** Build order below (§14) is dependency order, not a phased release.

## 2. Non-negotiable product rules (honesty contract)

1. **No fabricated evidence.** Every evidence quote shown to the user must exist verbatim (after normalization, §6) in their resume text. Quotes that fail verification are dropped; verdicts that lose all evidence are downgraded to `missing` and visibly flagged.
2. **The LLM never emits the score.** Score is a deterministic function of verdicts (§7). The UI can expand "how this number was computed" showing every requirement's contribution.
3. **No ghost-written experience.** Suggestions for `missing` requirements are actions (build X, learn Y, get cert Z) — never a resume bullet claiming experience the user doesn't have. Rewrite suggestions exist only for `partial` verdicts and must be grounded in the quoted evidence.
4. **Keyword stuffing doesn't pay.** A skill merely listed in a skills section with no demonstrated use is at best `partial`. This is prompted (§5) and tested by the eval (§12, the "keyword stuffer" gold pair).
5. **Failures are visible.** Anthropic outages/timeouts surface as a real error state with a retry button. Never a fake or degraded-silent result.

## 3. Stack & infrastructure

- **Next.js 15 (App Router) + TypeScript + Tailwind CSS 4** — same stack as qatar-dental-prep (`~/30 in 30 apps/qatar-dental-prep` is the reference for config/tooling conventions).
- **@anthropic-ai/sdk** — one client instance per process (module-level singleton), explicit `timeout` (60s), `maxRetries: 2`.
- **No database.** Nothing resume-derived is ever stored server-side. History is localStorage only.
- **Upstash Redis (free tier)** — rate-limit counters ONLY (keys = hashed IPs, no PII). Needed because in-memory rate limiting dies on serverless: each request may hit a fresh instance with a fresh counter.
- **pdfjs-dist** — client-side PDF text extraction. The PDF never leaves the browser.
- **Vitest** (unit) + **Playwright** (e2e via `superpowers:webapp-testing`).
- **Deploy:** Vercel. Domain: wire `gapmap.moizbuilds.com` subdomain (Moiz's Vercel DNS; give him the record values — the MCP Vercel account is not his real one).
- **Env vars (all fail closed — missing key = 503 with clear message, never open access):** `ANTHROPIC_API_KEY`, `UPSTASH_REDIS_REST_URL`, `UPSTASH_REDIS_REST_TOKEN`.

**Models:** stage 1 extract = `claude-haiku-4-5-20251001`. Stage 2 judge = Haiku first; the eval (§12) decides whether it upgrades to `claude-sonnet-5`. Model IDs live in ONE constants file. Cost reality: ~$0.012/analysis all-Haiku, ~$0.035 with Sonnet judge — abuse control matters more than model price.

## 4. Repo layout

```
gapmap/
  app/
    page.tsx                 # input screen (paste resume + JD, PDF upload)
    analysis/[id]/page.tsx   # results screen (reads from localStorage by id)
    history/page.tsx         # past analyses + comparison view
    api/analyze/route.ts     # the single API endpoint
    layout.tsx, globals.css, opengraph-image.png (static og image for landing)
  lib/
    pipeline/
      extract.ts             # stage 1: JD → requirements (LLM)
      judge.ts               # stage 2: requirements + resume → verdicts (LLM)
      verify.ts              # stage 3: quote verification (pure code)
      score.ts               # stage 4: deterministic scoring (pure code)
      pipeline.ts            # orchestrates 1–4; takes the Anthropic client as a PARAMETER (DI, testable without network)
    anthropic.ts             # the singleton client + model constants
    ratelimit.ts             # Upstash sliding window
    normalize.ts             # text normalization shared by verify.ts and UI highlighting
    validate.ts              # server-side input validation (caps, emptiness)
  types/analysis.ts          # THE shared contract — API, UI, and eval all import from here, never redeclare
  components/                # UI components (structure decided during frontend-design)
  eval/
    golden/                  # 10 gold pairs, one JSON file each
    run-eval.ts              # scoring script (npm run eval)
    ADJUDICATIONS.md         # disputed-case rulings
    RESULTS.md               # eval outcomes + the model decision
  tests/                     # vitest unit tests + playwright e2e
  docs/superpowers/specs/    # this file
```

Teaching comments per CLAUDE.md: plain-English block at the top of every file, `// CONCEPT:` for first-appearance concepts (DI, sliding window, NFKC, etc.), WHY-notes on pattern choices.

## 5. The pipeline

Two LLM calls + two pure-code stages. Both LLM calls use **forced tool use** (`tool_choice: {type: "tool", name: ...}`) so output arrives as a schema-validated `tool_use` block — no JSON-fence parsing. Still locate the `tool_use` block by iterating `response.content` (never assume `content[0]`; a thinking block may come first — this broke a real build). `max_tokens`: 2000 (extract), 8000 (judge).

### Stage 1 — Extract (LLM, Haiku)

User message (single message, no system prompt needed):

```
You are extracting hiring requirements from a job description, the way a hiring manager would build a screening checklist.

<job_description>
{JD_TEXT}
</job_description>

Extract every distinct requirement a hiring manager would actually screen for. Rules:
- One requirement per distinct qualification. Split bundled lists ("React, TypeScript, and GraphQL") into separate requirements when they'd be screened separately; keep genuinely single skills together.
- priority = "must_have" when stated as required/essential/minimum or clearly core to the role; "nice_to_have" when marked preferred/plus/bonus or clearly peripheral.
- category = one of: hard_skill, soft_skill, experience, education, domain, other.
- Ignore boilerplate (EEO statements, benefits, "fast-paced environment" filler).
- Order by importance, most important first. Maximum 30 requirements.
- Also extract jobTitle and company when present (null when absent).
```

Forced tool `record_requirements`, input schema mirroring `ExtractionResult` (§8). Code assigns each requirement a stable id `req_1..req_n` in order. If the model returns >30, truncate to 30 and set `requirementsTruncated: true` (UI shows a notice — no silent caps).

### Stage 2 — Judge (LLM, Haiku → eval may promote to Sonnet)

```
You are a rigorous, honest hiring screener judging a candidate's resume against a list of job requirements. Your judgments will be shown to the candidate with your quoted evidence, and every quote will be programmatically checked against the resume — a quote that is not a verbatim substring of the resume will be discarded and your verdict downgraded.

<resume>
{RESUME_TEXT}
</resume>

<requirements>
{REQUIREMENTS_JSON}   // [{id, text, category, priority}, ...]
</requirements>

For EVERY requirement id, return a judgment:

verdict rules:
- "match": the resume clearly demonstrates this requirement with concrete evidence (work done, results, duration).
- "partial": related or adjacent evidence exists but doesn't fully satisfy the requirement — OR the skill is merely listed (e.g. in a skills section) with no demonstrated use. Keyword mention alone is never a "match".
- "missing": no supporting evidence in the resume. When in doubt between partial and missing, and you cannot quote supporting text, choose missing.

evidence rules (critical):
- 1–3 quotes per judgment, each an EXACT verbatim substring copied character-for-character from the resume, max 300 characters each.
- Never paraphrase, merge, truncate mid-word, or "fix" the resume's wording inside a quote.
- "missing" verdicts have an empty evidence array.

suggestion rules:
- match: null, or one short optional strengthening tip.
- partial: a concrete, specific improvement — either a rewrite of one of the QUOTED bullets (grounded strictly in what the quote already claims — add specificity/metrics, never new claims), or a specific addition. If a rewrite, also return rewrittenBullet with the copy-ready line; otherwise rewrittenBullet is null.
- missing: a concrete action to close the gap (a specific project to build, a certification, a skill to learn and how to evidence it). NEVER write resume text claiming experience the candidate does not have. rewrittenBullet is always null for missing.

reasoning: one sentence explaining the verdict, plain language.
```

Forced tool `record_judgments`. Code validates: every requirement id present exactly once (missing ids get an auto-`missing` verdict flagged `judgeSkipped: true`; duplicate ids keep first).

### Stage 3 — Verify (pure code, `lib/pipeline/verify.ts`)

For each evidence quote: `normalize(resume).includes(normalize(quote))` (normalization in §6). Failed quotes are removed and counted. If a `match`/`partial` verdict ends with zero surviving quotes → verdict becomes `missing`, `evidenceRejected: true`, suggestion replaced with a generic honest line ("The analysis claimed support it couldn't quote from your resume, so this is treated as missing."). The UI renders the flag visibly (§10). Also store, per surviving quote, its character offsets in the ORIGINAL resume text (found via the normalization mapping, §6) so the UI can highlight real resume text rather than re-searching (store first-class fields, never re-derive).

### Stage 4 — Score (pure code, `lib/pipeline/score.ts`) — formula in §7.

Pipeline orchestration (`pipeline.ts`) runs 1→2→3→4 sequentially, accepts `(client: Anthropic, input: AnalyzeRequest)`, returns `AnalysisResult`. Any stage throwing surfaces as a 5xx with a typed error body — never a partial result.

## 6. Normalization (`lib/normalize.ts`)

One function used by the verifier AND the UI highlighter (one source of truth):

1. Unicode NFKC normalize
2. Map typographic characters to ASCII: ' ' → `'`, " " → `"`, – — → `-`, … → `...`, non-breaking space → space
3. Lowercase
4. Collapse every whitespace run (incl. newlines) to a single space; trim

Substring check runs on normalized text. To highlight in the original, `normalize.ts` also exposes a mapping variant that returns, for a normalized-text match range, the corresponding original-text range (build an index map during normalization — each normalized char remembers its source offset). Unit-test this hard: curly quotes, line-wrapped bullets, double spaces, unicode bullets (•), em-dashes.

## 7. Scoring — exact formula

```
value(match) = 1, value(partial) = 0.5, value(missing) = 0
weight(must_have) = 2, weight(nice_to_have) = 1

score = round( 100 * Σ(weight_i × value_i) / Σ(weight_i) )
```

- Verdicts downgraded by the verifier count as `missing` (the downgrade happens BEFORE scoring).
- Zero requirements extracted → typed error `NO_REQUIREMENTS` (JD probably wasn't a JD), not a score.
- Bands (label + one-line meaning, shown with the score): 85–100 **Strong fit** · 65–84 **Competitive** · 40–64 **Stretch** · 0–39 **Not yet** — labels are honest, no "you're almost there!" inflation on a 30.
- The breakdown view lists every requirement: its weight, its value, its contribution — the score is an audit trail.

## 8. Shared types (`types/analysis.ts`) — the one contract

```ts
export type Priority = "must_have" | "nice_to_have";
export type Category = "hard_skill" | "soft_skill" | "experience" | "education" | "domain" | "other";
export type VerdictLabel = "match" | "partial" | "missing";

export interface Requirement { id: string; text: string; category: Category; priority: Priority; }

export interface ExtractionResult {
  jobTitle: string | null; company: string | null;
  requirements: Requirement[]; requirementsTruncated: boolean;
}

export interface EvidenceQuote {
  quote: string;              // verbatim resume text (original casing — recovered via offsets)
  startOffset: number;        // char offsets into the ORIGINAL resume text
  endOffset: number;
}

export interface Judgment {
  requirementId: string;
  verdict: VerdictLabel;
  evidence: EvidenceQuote[];          // verified quotes only; [] for missing
  reasoning: string;
  suggestion: string | null;
  rewrittenBullet: string | null;     // partial-only, grounded in evidence
  evidenceRejected: boolean;          // verifier downgraded this verdict
  judgeSkipped: boolean;              // judge omitted this id; auto-missing
  droppedQuoteCount: number;          // quotes removed by the verifier
}

export interface ScoreBreakdown {
  score: number;                       // 0–100 integer
  band: "strong_fit" | "competitive" | "stretch" | "not_yet";
  counts: { match: number; partial: number; missing: number };
  rows: { requirementId: string; weight: number; value: number; contribution: number }[];
}

export interface AnalysisResult {
  id: string;                          // nanoid, generated server-side
  createdAt: string;                   // ISO, stamped server-side
  extraction: ExtractionResult;
  judgments: Judgment[];
  scoreBreakdown: ScoreBreakdown;
  judgeModel: string;                  // exact model id used — provenance for the eval story
}

export interface AnalyzeRequest { resumeText: string; jdText: string; }
export type AnalyzeError =
  | { error: "RATE_LIMITED"; retryAfterSeconds: number }
  | { error: "INPUT_TOO_LONG"; field: "resumeText" | "jdText"; max: number }
  | { error: "INPUT_EMPTY"; field: "resumeText" | "jdText" }
  | { error: "NO_REQUIREMENTS" }
  | { error: "SERVICE_UNAVAILABLE" }   // env missing / Anthropic down — 503
  | { error: "ANALYSIS_FAILED" };      // unexpected 5xx
```

API route, UI, eval script, and localStorage history ALL import these. Never redeclared.

## 9. API — `POST /api/analyze`

1. **Env check** — required env missing → 503 `SERVICE_UNAVAILABLE` (fail closed).
2. **Validate** (server-side; client mirrors for UX): `resumeText` ≤ 15,000 chars, `jdText` ≤ 10,000 chars, both non-empty after trim. Reject `NaN`-ish garbage by nature of string checks; enforce `Content-Type: application/json` and body size via Next config.
3. **Rate limit** — sliding window, **10 analyses/hour/IP**, keyed on SHA-256 of the platform-set client IP. IP source: `x-vercel-forwarded-for` first entry (set by Vercel's proxy; client-supplied values are stripped — this is why it's trustworthy, unlike raw `X-Forwarded-For`, which burned a previous build). Local dev fallback: `127.0.0.1`. Upstash `@upstash/ratelimit` slidingWindow. Over limit → 429 with `retryAfterSeconds`. Redis unreachable → 503 (fail closed, never open).
4. **Run pipeline** → 200 `AnalysisResult`.
5. Errors: Anthropic timeout/5xx → 503; anything else → 500 `ANALYSIS_FAILED`. Log server-side with stage name; never leak stack traces to the client.

No streaming for v1: the two calls take ~15–30s total; the UI covers the wait with a staged progress indicator (§10). (WHY: streaming forced-tool-use JSON adds real complexity for zero product value here — the result is only meaningful complete.)

## 10. UI/UX — screens & states

Run `superpowers:frontend-design` BEFORE writing any UI code and `web-interface-guidelines` after. Design bar: Linear/Apple-grade, distinctive typography, deliberate palette, generous whitespace, micro-interactions. Banned: Inter/system-font defaults, purple gradients, cookie-cutter cards. The results page is the product's face and the share card is its ad — spend the design budget there. (Aesthetic direction is the design skill's job; this spec only fixes structure and honesty affordances.)

**Input screen (`/`):** two inputs — resume (textarea + "upload PDF" affordance) and JD (textarea). Live char counters with the caps. PDF flow: extract client-side with pdfjs-dist → put text INTO the resume textarea for the user to eyeball/fix → garble heuristic (if letters/total ratio < 0.6 or extraction is empty: warning banner "This PDF didn't extract cleanly — please check the text below") — garbled text is never silently analyzed. Analyze button disabled until both inputs valid; on submit, staged progress ("Reading the job description… / Screening your resume… / Verifying every quote…") mapped to real pipeline stages via optimistic timing.

**Results screen (`/analysis/[id]`):** reads the `AnalysisResult` from localStorage by id (URL is shareable only on the same browser; that's fine — sharing happens via the card).
- **Score header:** big score + band label, match/partial/missing counts, jobTitle/company, the analyzed date. Expandable "How this score was computed" → the §7 formula and per-requirement contribution table. `judgeModel` shown in a subtle footer (provenance).
- **Requirement list, gaps first:** missing → partial → match. Each row: requirement text, priority chip (must-have visually heavier), verdict chip, evidence quotes rendered as highlighted excerpts of the actual resume (using stored offsets), reasoning line, suggestion. Partial rows with `rewrittenBullet` get a copy button labeled "suggested rewrite of your bullet". `evidenceRejected` rows show an explicit flag: "The model claimed evidence it couldn't quote — treated as missing." `requirementsTruncated` → banner "Showing the 30 most important requirements."
- **Share card:** client-rendered (html-to-image or canvas) PNG — score, band, counts, jobTitle, date, GapMap wordmark. ZERO resume/JD text. Download + copy-to-clipboard buttons. The card element must be fully rendered before export is enabled — ONE readiness check gates both buttons; on failure show a message, never a blank card.
- Empty/error states designed, not defaulted: rate-limited (shows retry-after), service-down (retry button), NO_REQUIREMENTS ("That doesn't look like a job description…").

**History (`/history`):** localStorage list (score, jobTitle, date). Entries store the full `AnalysisResult` + `resumeText` + `jdText` + `jdHash` (SHA-256 of normalized JD). **Comparison:** entries sharing a `jdHash` can be compared — score delta + per-requirement verdict changes (match↑/↓), keyed by requirement TEXT (ids differ across runs; note this in a teaching comment). "Re-analyze with my edited resume" pre-fills the input screen with the stored JD. React `key` for the results view = analysis id (identity-keyed UI, CLAUDE.md rule 8). localStorage quota errors caught → oldest entries evicted with a notice.

## 11. localStorage schema

```
gapmap:history:index     → string[] (analysis ids, newest first)
gapmap:analysis:{id}     → { result: AnalysisResult, resumeText, jdText, jdHash }
```

Versioned wrapper `{ v: 1, data: ... }` so a future schema change can migrate instead of crash. All reads defensive: JSON.parse in try/catch, invalid entries dropped.

## 12. Eval harness (ships WITH the app, not after)

**Gold set** (`eval/golden/*.json`, 10 pairs — realistic, written during build, expected verdicts reviewed by Moiz):

| # | Pair | What it tests |
|---|---|---|
| 1 | Strong fit (senior FE dev ↔ senior FE JD) | baseline matches |
| 2 | Career changer (sales → CS/AI role) | partial judgment on transferable skills |
| 3 | **Keyword stuffer** (skills-section-only resume) | stuffing = partial, never match |
| 4 | Zero overlap (chef ↔ ML engineer JD) | low score honesty; no fabricated evidence |
| 5 | Overqualified (staff eng ↔ junior JD) | matches don't over-penalize |
| 6 | Junior resume ↔ senior JD | experience-duration judgment |
| 7 | Non-tech (sales rep ↔ sales manager JD) | works beyond tech roles |
| 8 | Sparse one-page resume | missing ≠ hallucinated partial |
| 9 | Buzzword JD (vague requirements) | extraction ignores filler |
| 10 | Education mismatch (bootcamp ↔ degree-required JD) | education category judgment |

**Gold file format:** `{ name, resumeText, jdText, expected: [{ keywords: string[], expectedVerdict: VerdictLabel, note? }] }` — `keywords` fuzzy-match an expected requirement to an extracted one (case-insensitive; all keywords must appear in the requirement text). Solves the "extracted wording differs from expected wording" alignment problem.

**Scoring script (`npm run eval`, real API calls):** for each pair, run the full pipeline, then report: **extraction coverage** (% of expected requirements matched by keywords), **verdict accuracy** on matched pairs (overall + 3×3 confusion matrix), **fabrication rate** (% of quotes the verifier rejected — measures how often the honesty net catches the judge). Normalize before comparing; disputed cases (reasonable people could rule either way) get a ruling in `ADJUDICATIONS.md` and the gold file is corrected — don't let naive matching under/over-state accuracy.

**Model decision procedure:** run with judge=Haiku, then judge=Sonnet (`npm run eval -- --judge-model=claude-sonnet-5`). **Ship Haiku if verdict accuracy ≥ 90% AND no gold pair's score band is wrong (e.g. keyword stuffer landing "strong_fit"). Otherwise ship Sonnet for stage 2.** Record both runs + the decision in `eval/RESULTS.md` and the README — this measured decision is the portfolio story. Eval cost: ~10 pairs × 2 models × ~$0.01–0.03 ≈ under $1.

## 13. Testing

- **Vitest:** `normalize.ts` (curly quotes, whitespace, offset-mapping round-trips), `verify.ts` (fabricated quote → downgrade+flag; partial quote survival), `score.ts` (weights, bands, rounding, zero-requirements error), `validate.ts` (caps, empty, garbage), pipeline with a **mock Anthropic client via DI** (thinking-block-first responses, missing ids, duplicate ids, >30 requirements).
- **Playwright (`superpowers:webapp-testing`):** happy path paste→results with API mocked; error states (429, 503); history + comparison; share-card buttons gated on readiness. One manual end-to-end against the real API before claiming done.
- **Security exercise (per CLAUDE.md):** actually curl the deployed/local endpoint — oversized bodies, empty fields, spoofed `X-Forwarded-For` (must NOT bypass the limit), 11th request in an hour (must 429), missing env (must 503). Failing-first test for the rate limiter.

## 14. Build order (dependency order, one ship)

Each step ends with `/code-review`; delegate mechanical steps to the `builder` subagent, keep judgment (prompts, verify/score logic, security review, design direction) in the main thread.

1. Scaffold (create-next-app, Tailwind 4, vitest/playwright config mirroring qatar-dental-prep) + `types/analysis.ts` + constants.
2. `normalize.ts` + `verify.ts` + `score.ts` + `validate.ts`, unit-tested first (TDD — pure functions, ideal for it).
3. `anthropic.ts` + `extract.ts` + `judge.ts` + `pipeline.ts` with DI; unit tests with mock client.
4. `ratelimit.ts` + API route; security exercise (§13).
5. `superpowers:frontend-design` → input screen + results screen + states.
6. Share card + history + comparison.
7. PDF extraction + garble check.
8. Eval: gold set (Moiz reviews expected verdicts) → script → both model runs → decision → `RESULTS.md`.
9. `superpowers:webapp-testing` full pass + real-API smoke test.
10. Pre-flight checklist walk (all 9 CLAUDE.md questions, answered in writing in the PR/summary), README (with eval results + architecture diagram), deploy to Vercel, subdomain records for Moiz, teaching summary (3 bullets).

## 15. Pre-flight checklist — spec-level answers (verify at build time)

1. **Reload mid-action:** draft inputs are persisted to localStorage on change (debounced), so a reload mid-typing or mid-analysis keeps the user's text; the in-flight request itself is simply lost and can be resubmitted. Results/history survive reload by design.
2. **Fetch failure:** every fetch checks `res.ok`, typed error union, retry buttons. User's pasted text is never cleared on failure.
3. **Bad input:** server-side caps + emptiness (§9); NO_REQUIREMENTS path; garbled-PDF gate.
4. **Secrets:** all env fail closed; Anthropic key server-only (never `NEXT_PUBLIC`); Upstash token server-only.
5. **Cost:** rate limit + input caps + max_tokens caps + requirement cap (§9, §5).
6. **One source of truth:** `types/analysis.ts`; model ids in one constants file; normalization shared verifier↔highlighter; caps defined once, imported by client and server.
7. **Trust boundary:** IP from platform-set header only; hashed before use as Redis key; body validated server-side; localStorage reads defensive (user-editable = untrusted).
8. **Time & identity:** `createdAt` stamped server-side; results view keyed by analysis id; no countdowns exist.
9. **Derived data:** score derives only from post-verification verdicts; comparison matches on jdHash + requirement text (documented caveat).

No schema/DB migrations exist (no DB) — the code-and-DB sequencing rule doesn't apply; localStorage is versioned (§11).

## 16. Out of scope (explicitly)

Accounts/auth · server-side persistence or share links · multi-resume A/B against one JD · cover-letter generation · ATS-format checking · JD scraping from URLs (paste only — scraping adds legal/fragility surface) · i18n.
