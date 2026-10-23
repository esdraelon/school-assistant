# Packages — what exists vs what gets built

Repo granularity follows the estate's grain: a coarse repo per area,
with clean internal packages (the adapter's src-layout shape). Noah
creates the GitHub repos; ululuth remains the working origin, GitHub
the mirror.

## Repos

| Repo | Status | Holds |
|---|---|---|
| `school-info-adapter` | **Exists** (ululuth, live) | Extraction MCP |
| `school-assistant` | **Exists** (this repo) | Planning, personas, agent definitions, standards, orchestrator |
| `school-index` | **Build** | Corpus sync, text extraction, embedding, the index/search MCP |
| `school-calendar` | **Build** | Event source adapters, normalized events store, ICS feed |
| `school-core` | **Defer** | Shared kernel extraction — only when a second consumer exists (see below) |

## Exists today

- **`school` package (in school-info-adapter)** — domain types
  (Driver, Resource, Student, SourceKind), dedup/merge, SPI
  protocols, SQLite store, drivers (HAC, web, classroom-web), the
  AES-GCM vault, the FastMCP server, and the app/deploy kit. Live
  and soak-running.
- **Gateway estate on hal9k** — the auth token registry service
  (path-scoped bearer tokens), todo MCP, datenight MCP, and the
  grades MCP (legacy; still the digest v1 source, and the producer
  of the HAC cache the adapter reads — do not decommission the
  scraper with the serving layer; see the adapter's
  docs/LEGACY-DECOMMISSION.md).
- **Orchestration prototypes** — the digest crons and the soak jobs
  (health watch, v2 parallel run) are orchestrator-shaped tasks
  already running against the MCPs.
- **Hermes** — framework available on ululuth as an optional runtime
  for batch/agentic tasks (previous instances retired).

## Build

- **`school-index`**
  - `corpus_sync` — browser-driven Classroom corpus puller
    (docs/CLASSROOM-CORPUS.md); batch-shaped, Hermes candidate.
  - `doc_extract` — text extraction for Slides/Docs/Sheets/PDFs.
    Related to the adapter's deck SVG extraction, but a different
    job: raw text for indexing, not structured drivers.
  - `embed_index` — chunking + embedding + vector store. Embedding
    provider is a **strategy** (ONNX local vs Bedrock), selected by
    registry per config — per docs/RAG-EVALUATION.md and the coding
    standard's no-if/else-selection rule.
  - `index_mcp` — the search MCP app layer: passage search,
    topic→resources (with deep links), worked-example lookup.
- **`school-calendar`**
  - `event_sources` — adapters per source (calendar readers, the
    email-event extractor agent's output sink).
  - `events` — normalized event model + merge/dedup (same
    precedence/freshness discipline as the adapter merge).
  - `calendar_feed` — ICS feed service + event query MCP.
- **`school-assistant` (this repo, code to come)**
  - `orchestrator` — scheduler, global config, notification state,
    rule engines (lead-time, low-grade, makeup detection), sub-agent
    dispatch.
  - `tutor` — tutor/study policy: similar-problem generation, the
    similarity checker, pointer composition, study planner, paced
    repetition scheduler (rides the todo MCP).
  - `mcp_clients` — thin typed clients for the estate's MCPs, so no
    agent hand-rolls HTTP (tonight's smoke scripts are the evidence
    for why this exists).
  - `personas/` + `skills/` — definitions the orchestrator loads
    (already started).

## Defer

- **`school-core`** — the adapter's domain/merge/SPI modules are the
  obvious shared kernel, and `school-index` will want the resource
  types. Rule: don't extract for one consumer. When the index (or
  calendar) genuinely needs the types, extract then — the adapter's
  modules were built separable for exactly this.
- **A shared MCP server kit** — the gateway pattern (root-path
  serving, host allowlist, bearer middleware, token registration)
  is now deploy-proven but exists in exactly one codebase. Codify
  it in the coding standard first; extract a kit when the second
  MCP is built from it (school-index will be the test).
