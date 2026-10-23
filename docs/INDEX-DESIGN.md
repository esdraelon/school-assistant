# Design: school-index v1

Status: draft, 2026-10-23. Plan + checklist: docs/PROJECT-INDEX.md.
Standards: docs/CODING-STANDARD.md. Every artifact below is a
schema in a store (data discipline, docs/ARCHITECTURE.md).

## Components

- **`corpus_sync` — harvester.** Consumes the school MCP's
  `get_resources` (the contract — never its database) and downloads
  or exports each reachable resource into the corpus store.
  Incremental by content hash + source identity; deletions become
  tombstones. Scheduled batch work; a Hermes task is a candidate
  runtime.
- **`doc_extract` — text extraction.** A registry: format →
  extractor (published Slides via the SVG accessibility approach
  the adapter proved; Docs/Sheets via export; PDF via text layer;
  HTML/plain passthrough). Per-item failure containment — one bad
  file never fails a run, and the failure is a record.
- **`embed_index` — chunk, embed, store vectors.** Structure-aware
  chunking where the format gives structure (slide / heading
  boundaries). Embedding is a **strategy registry** (ONNX bge
  default; Bedrock as the benchmark arm). Vectors in sqlite-vec.
  Only changed chunks re-embed.
- **`index_mcp` — the search MCP.** FastMCP app per the deployment
  standard, read-only over the store:
  - `search_passages(query, course?, limit)` → ranked passages,
    each with its item's title, course, and **deep link**.
  - `resources_for_topic(topic, course?)` → ranked resources with
    deep links — the tutor's pointer source.
  - `worked_examples(topic)` → example-like passages for composing
    similar problems.
  Every result carries provenance (item, source, deep link) —
  consumers cite records, not recollections.

## Schemas (SQLite on hal9k)

- `corpus_items` — id, source_kind, source_ref (adapter resource
  id), student, course, title, deep_link, origin_url, format,
  content_hash, fetched_at, status (ok | failed | tombstoned),
  error.
- `item_text` — item_id, extracted text, extractor id,
  extracted_at.
- `chunks` — id, item_id, ordinal, heading (nullable), text.
- `embeddings` — chunk_id, model_id, vector (sqlite-vec table).
  model_id is recorded per vector; mixing models in one index is
  invalid — a model change is a full re-embed maintenance op.
- `sync_state` — source, watermark, last_run_at, last_status,
  counts (the adapter's per-source health shape).
- `benchmark_runs` — query-set version, arm (provider + chunking
  settings), per-query scores. Benchmark results are records, not
  a chat message (data discipline).

## Flows

Harvest (timer) → extract → chunk → embed deltas → store.
The MCP serves read-only from the store; consumers (tutor, study
planner) bind to the MCP tools. Corpus expansion (Classroom
profile channel, after profile birth) adds a source_kind to
`corpus_sync` — no other component changes.

## Deployment

Container on hal9k per the deployment standard; gateway route
`/mcp/index/`; own token, client `index-mcp`, paths exactly
`["/mcp/index"]`; store on a mounted volume owned by the
container uid; sync on a systemd timer.

## Open within the design

- Whether `get_resources` pagination suffices at corpus scale —
  M0 verifies against the live adapter (hundreds of resources;
  expected fine).
- Chunk size/overlap defaults — M5's benchmark may add a chunking
  arm if pointer accuracy looks chunk-bound.
