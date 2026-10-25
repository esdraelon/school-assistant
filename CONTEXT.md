# Context — school-assistant

_The file an agent reads first. Kept current by whoever changes the
state it describes. Last updated 2026-10-24._

## What this is

An educational orchestration system for one household, in layers:
a data-extraction MCP (live), supporting MCPs for search/calendar/
todo, and an orchestration agent running two personas — Guardian
(parents) and Student (tutor/study, on Sam's own Muse agent).
Full picture: README.md, docs/ARCHITECTURE.md, personas/.

## Where things live

- **Hosts:** everything durable is on **hal9k** (hal9k.ululuth.com,
  SSH user `haliffer`, NOPASSWD sudo; Docker via sudo).
- **Repos (origin: hal9k `~/git/`, GitHub mirrors pending):**
  - `school-info-adapter` — extraction MCP. Clone: `~/school-info-adapter`.
  - `school-assistant` — this repo. Clone: `~/school-assistant`.
  - `school-index` — corpus/index + search MCP. Clone:
    `~/school-index`; corpus root `~/school-index-data` (corpus.db,
    benchmark.db, model files).
  - `school-calendar` — normalized events store + ICS feed (M0/M1
    done 2026-10-24; collectors not yet built). Clone:
    `~/school-calendar`.
- **Live service:** container `haliffer-school-mcp` on hal9k;
  gateway route `https://haliffer.ululuth.com/mcp/school/`;
  scrape timer every 6h at :15; deploy dir `deploy-run/` inside
  the adapter clone; production deck registry excludes the
  tutorial deck.
- **Live service (index):** search MCP at gateway route
  `https://haliffer.ululuth.com/mcp/index/`; corpus pipeline
  (harvest → extract → chunk/embed) on a systemd timer every 6h
  at :45 against `~/school-index-data`. Serving code is index
  main `bd742fd` (2026-10-24 deploy); the timer clone
  `~/school-index` sits at merged main.
- **Gateway auth:** `/var/www/haliffer/auth/tokens.json` —
  sha256(token) → {client, paths}, checked per request by the
  `haliffer-auth` service behind nginx `auth_request`. The school
  token is client `school-mcp`, paths `["/mcp/school"]`, presented
  alone as the Bearer; the app re-checks it. The file is a
  single-file bind mount into the auth container: edit in place or
  restart the container after replacing it.
- **Digest:** v1 (grades MCP) is delivered daily ~18:00. The
  adapter is in **soak** (started 2026-10-22): daily health watch
  + Mon–Fri v2 parallel drafts; cutover after the parallel run's
  final 3 consecutive clean days (docs in the adapter repo:
  docs/SOAK-PLAN.md, deploy/DIGEST_V2.md).

## State of play

### 2026-10-24 — pagination audit, merges, deploy

- **Pagination verdict (Phase 0 audit): the M1 harvest was
  INCOMPLETE.** Adapter collection tools default to limit 200,
  silent truncation, no cursor, no total. Live probe found 197
  distinct resources (not 99) and 562 true drivers (not 200);
  5 items were falsely tombstoned after drifting past the
  title-sorted cap. Standing rule (adapter docs, school-index
  docs/HARVEST-PAGING.md, this repo's CODING-STANDARD.md):
  every collection tool documents limit + ordering + paging;
  every consumer pages to exhaustion and proves total + bounds
  in tests; a single default-limit call is a gate failure.
- **Harvest fix deployed:** corpus_sync pages `get_resources`
  to exhaustion (probe + course × kind partitions, deduped,
  exhaustion proof; a partition hitting exactly 200 fails
  loudly). Corpus census after the deploy pipeline run:
  **209 distinct, 0 tombstoned, 636 chunks** (was 99 / 371 at
  M1). Failures are 90 Google sign-in walls + 7× 401 + 2× 404 —
  the sign-in set is the profile-birth unlock set. Benchmark
  re-run on the enlarged corpus holds hit@3 **1.00** on both
  tools; hit@1 dipped (near-neighbors steal #1) — recorded,
  not retuned (frozen contract).
- **Index fixes deployed with the harvest fix:** title fallback
  (course + first-chunk heading + 80-char snippet, at
  serialization — no wire change; benchmark exactly
  reproduced) and a deterministic why-wrong reformulation
  helper (validated live: misconception phrasings flip weak →
  strong; no benchmark regression, no benchmark query matched
  a frame).
- **Merges 2026-10-24 (all fast-forward to main, gates green
  on both machines):** index main `bd742fd` (harvest fix +
  index fixes); adapter main `653f556` (pagination contract
  doc); assistant main `995f912` (pagination rule +
  docs/TUTOR-SIMILARITY-CHECK.md); calendar main `97f7f23`
  (M0 scaffold + M1 inventory; repo created by the merge).
- **Calendar M1 (inventory only):** operative calendar is
  Smithborough (shared family calendar, 60 events, paging
  proven); the "configured 8th grade calendar" was NOT found
  as a distinct calendar; director email is a first-class
  source (today's District Band Auditions exist in director
  mail but nowhere on Smithborough); agenda decks are
  DRIVERS, not events. Schema draft carries the
  CALENDAR-EVENTS.md contract + a reserved
  `validation_verdict` provenance field (ingestion-validator
  open item, unscoped).
- **Tutor similarity-check DESIGN (no code):**
  docs/TUTOR-SIMILARITY-CHECK.md recommends deterministic
  symbolic normalization (template signature + numeric
  multiset) over embeddings — embeddings cannot separate a
  desired similar problem from a forbidden restatement.
  Building the tutor is NOT authorized by the design doc.
- **Still open, in order:** profile birth (supervised session
  with Noah + Sam — unlocks the 90 sign-in-walled files,
  then benchmark re-run); digest v2 query-path fix (same
  200-cap affects v2 resource attachment — needs its own
  milestone before cutover reliance); digest v2 cutover
  (~2026-10-29 if parallel days stay clean); calendar M2
  collectors; tutor build decision; RAG stack (Noah's call);
  contract follow-ups (q27 breadth-arm over-claim, passages
  threshold headroom 0.0006, small-item resources ceiling);
  GitHub mirrors (Noah creates the repos).
- **Digest v2 soak:** unchanged — v1 delivered daily ~18:00
  until the cutover bar (5 school days parallel, final 3
  consecutive clean).

### 2026-10-23

- Adapter M0–M7.5 built, deployed, verified through the gateway
  (476 unit + 17 system tests; the full suite runs green ON hal9k
  including real-Chromium Classroom tests).
- **school-index v1 COMPLETE (M0–M6, 2026-10-23)** and live
  (above). M5 benchmark over the live corpus (33 labeled
  queries): ONNX bge-small + sqlite-vec, resources hit@3 0.93 /
  passages 1.00 at ~30 ms; Bedrock arm BLOCKED (no AWS
  credentials on hal9k — provider built, one config away). M6
  closed the two M5 findings: **banded item aggregation**
  (benchmark re-run `onnx-production-v1.0-20261023T161119188397`:
  resources hit@1 0.97 / hit@3 **1.00** / MRR 0.977, passages
  unchanged) and **`match_quality`** on every tool envelope
  (absent probes 4/4 weak; scored tops strong 28/29 resources,
  27/29 passages). Response shapes are frozen as **tutor
  contract v1** (school-index repo: docs/TUTOR-CONTRACT.md +
  contract fixtures). The RAG stack decision remains **Noah's**
  — recommendation (keep ONNX + sqlite-vec) and evidence in
  docs/RAG-EVALUATION.md. **What remains:** profile birth
  expands the corpus (Classroom channel into corpus_sync;
  expect the calibration constants to be re-derived against the
  larger corpus); two recorded issues — why-is-X-wrong phrasings
  under-score (tutor-side concept reformulation is the
  mitigation) and title ambiguity in the corpus (pointers are
  unambiguous by deep link). See docs/PROJECT-INDEX.md closeout.
- CLASSROOM_WEB source awaits **profile birth**: a supervised
  session with Noah + Sam on hal9k (vault enrollment rides along).
  That also unblocks the corpus pull (docs/CLASSROOM-CORPUS.md).
- Persona/capability design settled 2026-10-23 (docs/DECISIONS.md),
  including: config global for now; index scope = all Classroom
  docs with deep links preserved; events from calendars + email.
- Open decisions: RAG stack (benchmarked 2026-10-23 —
  recommendation in docs/RAG-EVALUATION.md, Noah's call);
  tutor BUILD (design doc 2026-10-24 recommends deterministic
  symbolic normalization; building is a separate decision);
  digest v2 query-path fix + cutover; calendar M2 collectors;
  contract calibration follow-ups (see 2026-10-24 notes);
  GitHub mirrors (Noah creates the repos); Hermes scope per
  sub-agent. RESOLVED since 2026-10-23: similarity-check
  mechanism (design recommendation), events source inventory
  (M1), why-wrong under-scoring + title ambiguity (both
  fixed 2026-10-24, see above).

## Rules that bite (details in docs/CODING-STANDARD.md)

- MCPs stay single-purpose; policy lives in the orchestration layer.
- **Personas read stores, not memory.** All operating evidence is a
  schema-validated record in a store on hal9k; interpretation
  happens once, at ingestion (docs/ARCHITECTURE.md — data
  discipline). The agent-coordination mechanism for external
  sources is deferred; the store/schema contract is not.
- Student references always cite the original Classroom deep link.
- Tutor never answers the assigned problem — guidance, similar
  problems, why-is-X-wrong, pointers.
- Freshness is data: surfaces state how fresh their data is.
- **Pagination & coverage (standing rule, 2026-10-24):** no
  consumer relies on a single default-limit call. Page to
  exhaustion; prove total count + min/max bounds in tests.
- **Orchestration paradigm, always (reaffirmed 2026-10-24):**
  plan → design → milestone checklist → code → hygiene gates
  (coverage, mypy strict, LOC cap, system tests) → dispatcher
  re-runs gates independently → stacked branches, fast-forward
  to main only. It applies to **bug-hunt spikes too** — a spike
  gets a scaled plan + checklist + gates, never a shortcut
  around them. (skills/development-orchestration/SKILL.md)
