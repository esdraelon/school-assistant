# Architecture

## Layers

```
Surfaces          Noah's Muse (Guardian)      Sam's Muse (Student)
                        \                    /
Orchestration     Educational orchestration agent
                  (schedules, config, state; spawns sub-agents:
                   notifier · tutor · study planner · indexer)
                        |
MCP resources     extraction · index/search · calendar · todo
                        |
Data (ululuth)    HAC cache · agenda decks · Classroom profile +
                  corpus · vector index · state stores
```

The MCPs are deliberately **not monolithic**: each does one job and
exposes it over the gateway pattern (own route, own token, path
scoped). Personas and policy live in the orchestration layer, never
in an MCP.

## Data discipline

The system's hard rule (Noah, 2026-10-23):

- **All operating evidence and artifacts live in discrete, concrete
  schemas** — typed records in stores on hal9k, validated at the
  write boundary. The adapter's domain layer is the reference
  implementation: nothing enters its store unvalidated.
- **Agents may sit on the ingestion path.** Extraction from prose,
  pages, and decks is legitimately agentic and fuzzy — but an
  agent's output becomes operational only as a schema-validated
  record. Ingestion is where interpretation happens, and it
  happens **once**.
- **Runtime personas read stores, through MCP tools.** Never raw
  sources, never the model's own memory or recollection of a prior
  run, never a reinterpretation of a source at read time. Reads
  are lookups against records treated as concrete and correct.
- **Provenance and freshness are properties of the record**
  (source, captured-at, source health) — so anything a persona
  states traces to the record it came from, and staleness is
  computed from the store, not recalled.

Corollary: because personas bind to stored schemas rather than to
how records got produced, coordination mechanisms between agents
(orchestration, dispatch, messaging) are interchangeable
implementation details — chosen, and changeable, later.

## MCP inventory

| MCP | Job | Status |
|---|---|---|
| school-info-adapter | Extraction: drivers, resources, source health from HAC + agenda decks + team site (+ Classroom web after profile birth) | **Live** 2026-10-22, `haliffer.ululuth.com/mcp/school/` |
| index / semantic search | Chunked, embedded Classroom corpus; passage search, topic→resource lookup | Planned — docs/RAG-EVALUATION.md |
| calendar | ICS feed of due dates that matter + events/extracurriculars | Planned — docs/CALENDAR-EVENTS.md |
| todo | Household tasks; study plans + repetition sessions ride it | Exists on the gateway |

## Key flows

**Indexing.** Corpus sync (docs/CLASSROOM-CORPUS.md) lands originals
+ metadata on hal9k → text extraction → chunking → embedding →
vector index. The index MCP answers: passages about X; which
resources cover topic T (with deep links); worked examples for T.

**Notification.** Orchestrator polls drivers on a schedule → applies
lead-time / low-grade / makeup rules (global config) → dedups
against notification state → delivers via the persona surfaces.

**Tutoring.** Sam asks → tutor sub-agent searches the index → either
a pointer (resource + deep link + section) or a worked similar
problem (generated from the topic's indexed examples, similarity-
checked) → why-is-X-wrong handled against the same material.

**Study.** QUIZ/TEST driver inside the planning horizon → study
planner builds resource list + practice set + repetition schedule →
sessions as todo items → round results update per-topic state.

## State placement

Everything durable on hal9k: corpus, index, configs, notification
log, repetition state. Agents (Muse or Hermes) are stateless
workers over that state and are replaceable without data loss.
