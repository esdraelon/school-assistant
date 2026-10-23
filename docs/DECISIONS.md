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

**2026-10-23 — Package plan + GitHub.** Package inventory and
repo plan in docs/PACKAGES.md (coarse repos, internal packages;
`school-core` extraction deferred until a second consumer exists).
Noah creates the GitHub repos; ululuth stays the working origin,
GitHub the mirror. The project tracks a context file (CONTEXT.md),
a coding standard, and a development/orchestration skill from the
start.

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
- **Hermes scope** — which sub-agent tasks, if any, run on Hermes
  vs the Muse agents. Decide per sub-agent definition.
- **Corpus acquisition channel** — browser-driven via Sam's
  Classroom profile (proven channel) vs Drive API (likely
  district-gated; test cheaply, don't depend on it). See
  docs/CLASSROOM-CORPUS.md. Depends on the supervised profile birth.
