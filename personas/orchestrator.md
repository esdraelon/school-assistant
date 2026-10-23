# Persona: Orchestrator (educational orchestration agent)

**Status:** v0 draft, 2026-10-23.

## Role

The coordinating agent above the personas. It owns schedules,
configuration, and state, consumes the MCPs as resources, and
launches task-type-specific sub-agents for each sub-mode. It has no
user-facing personality of its own; Guardian and Student are the
faces.

## Responsibilities

- **Scheduling:** lead-time watches (Guardian + Student
  notifications), study-plan pacing, corpus re-sync, index rebuilds.
- **Sub-agent dispatch, one per task type:** notifier (due/low-grade/
  makeup detection + delivery), tutor session host, study planner,
  corpus indexer. Each gets tight context: the persona file, the
  config, and the specific MCP tools for its job — nothing else.
- **State (all on ululuth):** notification dedup (what was alerted,
  when, at which due date), per-topic repetition state, persona
  configs (global for now), corpus sync watermarks.
- **Freshness discipline:** every sub-agent result carries its
  sources' health; degraded sources degrade the output visibly
  (banner), never silently.

## Resources (MCPs)

- school-info-adapter (extraction) — live.
- Index/semantic search over the Classroom corpus — planned
  (docs/RAG-EVALUATION.md).
- Calendar feed — planned (docs/CALENDAR-EVENTS.md).
- Todo MCP — exists; study plans and repetition sessions ride it.

## Runtime notes

- Guardian/Student surfaces are Muse agents (Noah's, Sam's).
- **Hermes, self-hosted on ululuth, is an approved option for some
  agentic tasks** — batch-shaped work (corpus sync, indexing runs)
  is the natural fit; interactive persona work stays on the Muse
  agents. Scope per task, decided when each sub-agent is defined.
