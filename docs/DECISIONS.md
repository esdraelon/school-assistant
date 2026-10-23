# Decisions

## Settled

**2026-10-23 — Persona model.** Two major personas (Guardian,
Student) plus the orchestration agent; sub-modes: notifications,
tutor, study. Basis for the agent definitions.

**2026-10-23 — Student surface.** Sam gets his own Muse agent.
Hermes, self-hosted on ululuth, may be used for some agentic tasks
(batch-shaped work is the natural fit).

**2026-10-23 — Config is global for now.** One notification lead
time, one low-grade threshold. No per-course/per-kind config until
experience demands it.

**2026-10-23 — Events from calendars + email.** Extracurricular
dates/times assemble from various calendars and emails; the
configured 8th grade calendar carries some of it. This requires
coordinating with other agentic sources (calendar owners, email
readers) — see docs/CALENDAR-EVENTS.md.

**2026-10-23 — Index scope is ALL Classroom docs.** Pull the full
corpus local for agentic use. Student references always cite the
original Classroom deep link, never the local copy — see
docs/CLASSROOM-CORPUS.md.

**2026-10-23 — MCPs stay single-purpose.** The school MCP remains a
data-extraction layer; indexing, calendar, and todo are separate
MCPs consumed by the orchestration agent.

**2026-10-23 — Separate project.** This work lives in
`school-assistant` on ululuth, not inside the adapter repo.

**2026-10-23 — Spaced repetition, two modes.** Occasional
prompting (ad hoc retrieval questions on the notification channel,
low friction) vs focused study sessions targeting **85% measured
mastery** per topic. Mastery and repetition state are stored per
topic, per the data discipline.

**2026-10-23 — Student reminders scaffold executive function.**
Student-facing notifications are prompts ("you might check if
something is due tomorrow"), not declarations — the persona knows
the record and withholds it on purpose. Specificity graduates with
engagement and deadline criticality; prompt/response history is
stored state. Guardian notifications stay declarative.

**2026-10-23 — Data discipline.** Agents may sit on the ingestion
path, but all operating evidence and artifacts are stored in
discrete, concrete, schema-validated schemas on ululuth. Runtime
personas consume those stores as concrete ground truth — no
operating from agent memory, no reinterpretation at read time.
Interpretation happens once, at ingestion; provenance and
freshness are properties of the record. See docs/ARCHITECTURE.md.

**2026-10-23 — Package plan + GitHub.** Package inventory and
repo plan in docs/PACKAGES.md (coarse repos, internal packages;
`school-core` extraction deferred until a second consumer exists).
Noah creates the GitHub repos; ululuth stays the working origin,
GitHub the mirror. The project tracks a context file (CONTEXT.md),
a coding standard, and a development/orchestration skill from the
start.

**2026-10-23 — Index boundary: consume the adapter's MCP contract.**
`school-index`'s `corpus_sync` consumes resource types via the
school adapter's `get_resources` MCP contract (never its database);
`school-core` extraction is NOT triggered — no second consumer of
the domain types exists yet. Evidence (live gateway probe,
2026-10-23): `get_resources("sam")` returned all 200 resources in
a single call, no pagination — kinds TUTORIAL 102, LINK 57,
DOCUMENT 21, NOTES 20 — each item carrying student, course, title,
kind, url, source_kind, and scraped_at, with a url on all 200
(docs.google.com 129, drive.google.com 44, www.ixl.com 19,
sites.google.com 7, www.google.com 1). This also answers
INDEX-DESIGN's open pagination question for v1 scale. Revisit
extraction when a second type consumer actually appears.

## Open

- **RAG stack** — embeddings (Bedrock vs local ONNX) and vector
  store (sqlite-vec vs pgvector vs Qdrant vs …). Evaluation in
  docs/RAG-EVALUATION.md; decision is Noah's.
- **Similarity-check mechanism** — how the tutor proves a "similar"
  problem isn't the assigned problem restated (numeric/structural
  comparison vs a checker pass). Needs a design before tutor ships.
- **Events source inventory** — which calendars exist, who owns
  them, what's already on the 8th grade calendar; email-derived
  event extraction scope. See docs/CALENDAR-EVENTS.md.
- **Coordination mechanism** — DEFERRED (Noah, 2026-10-23: cross
  that bridge later). The shared events store + normalized schema
  in docs/CALENDAR-EVENTS.md are settled as part of the data
  discipline; how collectors/agents are dispatched and coordinated
  is the deferred part, and the discipline is what makes deferring
  it cheap.
- **Hermes scope** — which sub-agent tasks, if any, run on Hermes
  vs the Muse agents. Decide per sub-agent definition.
- **Corpus acquisition channel** — browser-driven via Sam's
  Classroom profile (proven channel) vs Drive API (likely
  district-gated; test cheaply, don't depend on it). See
  docs/CLASSROOM-CORPUS.md. Depends on the supervised profile birth.
