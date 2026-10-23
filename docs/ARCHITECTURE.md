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
