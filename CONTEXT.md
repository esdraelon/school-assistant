# Context — school-assistant

_The file an agent reads first. Kept current by whoever changes the
state it describes. Last updated 2026-10-23._

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
- **Live service:** container `haliffer-school-mcp` on hal9k;
  gateway route `https://haliffer.ululuth.com/mcp/school/`;
  scrape timer every 6h at :15; deploy dir `deploy-run/` inside
  the adapter clone; production deck registry excludes the
  tutorial deck.
- **Live service (index):** search MCP at gateway route
  `https://haliffer.ululuth.com/mcp/index/`; corpus pipeline
  (harvest → extract → chunk/embed) on a systemd timer every 6h
  at :45 against `~/school-index-data`.
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

## State of play (2026-10-23)

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
  recommendation in docs/RAG-EVALUATION.md, Noah's call),
  similarity-check mechanism, events source inventory, Hermes
  scope per sub-agent.

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
