# school-assistant

Educational orchestration for the household, built on top of the
**school-info-adapter** extraction layer (repo: `school-info-adapter`
on ululuth, live at `https://haliffer.ululuth.com/mcp/school/`).

Status: **planning** (2026-10-23). This repo holds the planning docs,
the persona definitions, and the plans for the additional MCPs and
packages the system needs. Code lands here (or in sibling repos) once
the plans settle.

## Shape of the system

- **Extraction stays separate.** The school MCP is a data-extraction
  layer — drivers (due items), resources, source health — and nothing
  more. No persona logic, no notifications, no tutoring in the MCP.
- **Supporting MCPs, one job each:** semantic search / indexing over
  the Classroom corpus, a calendar feed, and the existing todo MCP.
- **One orchestration agent** owns the personas and the scheduling,
  and launches task-type-specific sub-agents per sub-mode (notifier,
  tutor session, study planner, indexer).
- **Surfaces:** Guardian delivers to Noah (and Diana). The student
  persona runs on **Sam's own Muse agent**. Hermes, self-hosted on
  ululuth, is an option for some agentic/batch tasks.
- **All durable state lives on ululuth** (hal9k): configs, the
  corpus, the index, notification/repetition state. Agents are
  replaceable; the data is the system of record.

## Principles

- **Pointers cite the original.** The corpus and index are local
  working copies for agents. Anything shown to the student references
  the original Google Classroom deep link, never the local copy.
- **Tutor never answers.** Guidance, similar problems, and
  why-is-this-wrong explanations — never the assigned problem's
  solution. This is policy enforced by design, not just prompting
  (see personas/student.md).
- **Honest freshness.** Inherited from the adapter: every surface
  states how fresh its data is instead of presenting stale data as
  current.
- **Config is global for now.** One lead time, one low-grade
  threshold. Per-course/per-kind refinement only if experience
  demands it.

## Map

- `personas/` — the persona definitions (the basis for the agent
  definitions): `guardian.md`, `student.md`, `orchestrator.md`.
- `docs/ARCHITECTURE.md` — layers, MCP inventory, data flows.
- `docs/DECISIONS.md` — settled decisions and the open ones.
- `docs/RAG-EVALUATION.md` — embeddings + vector store evaluation
  (Bedrock vs ONNX vs vector DBs) for the corpus index.
- `docs/CLASSROOM-CORPUS.md` — pulling the full Classroom doc corpus
  local, with deep links preserved.
- `docs/CALENDAR-EVENTS.md` — events from calendars + email, and the
  guardian calendar feed.
