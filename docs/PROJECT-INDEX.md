# First project: school-index v1

**Decision (2026-10-23):** the index is the first build. Tutor and
study block on it; Guardian does not (Guardian alerts ride the
digest v2 cutover already soaking, then become the first
orchestrator package). Draft plan — milestone dispatch starts once
the GitHub repos exist and Noah gives the go.

## Scope

**In:** corpus from sources reachable today (the adapter's
resources: published agenda decks, team-site + resource links, the
tutorial corpus), text extraction, chunking + embedding, the
index/search MCP on the gateway pattern, and the RAG benchmark.

**Out (v1):** the full Google Classroom corpus (needs profile
birth — slots in as corpus expansion, not a redesign); Guardian
notifications; any tutor behavior (consumers come after the MCP).

## Packages (per docs/PACKAGES.md)

- `corpus_sync` — resource harvester: pulls files/exports for the
  adapter's resources into the corpus store (original + extracted
  text + metadata incl. the Classroom/source deep link).
- `doc_extract` — Slides/Docs/Sheets/PDF text extraction.
- `embed_index` — chunking, embedding (strategy registry: ONNX
  default, Bedrock as the benchmark arm), sqlite-vec store.
- `index_mcp` — search MCP: passage search; topic → resources
  (deep links); worked-example lookup.

## Milestones — work checklist

Design: docs/INDEX-DESIGN.md. Each milestone: one agent, stacked
branch, discrete commits; the dispatcher re-runs verify.sh before
accepting (coding standard).

### M0 — Skeleton + standards
- [ ] Repo scaffold: src layout, pyproject, `scripts/verify.sh`
      (tests, ≥90% coverage, mypy strict, 500-LOC cap) green on an
      empty package.
- [ ] Verify `get_resources` against the live adapter (pagination,
      counts, deep-link fields) — evidence for the boundary call.
- [ ] Decide: consume resource types via the adapter's MCP contract
      or extract `school-core`. Record in DECISIONS.md.
- [ ] **Gate:** verify.sh green; decisions recorded.

### M1 — Corpus sync (reachable sources)
- [ ] `corpus_items` + `sync_state` schemas; store module.
- [ ] Harvester over the adapter's resources: download/export,
      content hash, idempotent re-runs (second run = no changes).
- [ ] Tombstones for vanished resources; per-item failure records.
- [ ] Fixture corpus from real resource shapes; system test mocks
      only the HTTP edge.
- [ ] **Gate:** full harvest of Sam's current resources completes;
      re-run is a no-op; coverage ≥90%.

### M2 — Extraction
- [ ] Extractor registry (format → strategy); Slides (SVG-a11y),
      Docs/Sheets export, PDF text layer, HTML/text passthrough.
- [ ] `item_text` populated per item; per-item failure containment
      with the failure stored, not logged-and-lost.
- [ ] **Gate:** every ok corpus item has text or a failure record;
      no silent skips in the run report.

### M3 — Chunk + embed (ONNX)
- [ ] Structure-aware chunker; `chunks` schema.
- [ ] Embedding strategy registry; ONNX (bge) arm working on
      hal9k (ARM); sqlite-vec store; `embeddings` keyed by chunk +
      model_id.
- [ ] Incremental: only changed chunks re-embed; model change =
      explicit full re-embed op.
- [ ] **Gate:** index builds from the real corpus; a changed item
      re-embeds only its own chunks.

### M4 — Index MCP + deploy
- [ ] Three tools per the design (`search_passages`,
      `resources_for_topic`, `worked_examples`); every result
      carries title + course + deep link + provenance.
- [ ] Deploy per the runbook: container, `/mcp/index/` route, own
      token (`index-mcp`, path-scoped), volume ownership, timer.
- [ ] Smoke through the public gateway route (real protocol);
      sibling routes undisturbed.
- [ ] **Gate:** live queries return traceable results; negative
      auth checks pass; deploy notes land in deploy/README.md.

### M5 — RAG benchmark
- [ ] ~30-query labeled set from the real corpus (topics, expected
      resources/passages), versioned as data.
- [ ] Arms: ONNX+sqlite-vec vs Bedrock (same chunking); optional
      chunking arm if accuracy looks chunk-bound.
- [ ] Results stored as `benchmark_runs` records; summary into
      docs/RAG-EVALUATION.md; stack decision recorded in
      DECISIONS.md (Noah's call).
- [ ] **Gate:** benchmark reproducible from the stored records.

### M6 — Tutor-facing shapes
- [ ] Walk personas/student.md flows (pointer answers, similar-
      problem composition inputs, study-plan resource lists)
      against the live MCP; file gaps as issues, fix what's cheap.
- [ ] Response shapes frozen as the tutor contract (fixtures).
- [ ] **Gate:** every student-persona query pattern returns what
      the persona needs, deep links intact.

## Acceptance

Gates green per the coding standard; index answers from the public
gateway route; every result traceable to an original via its deep
link; benchmark done and the RAG decision made with data.

## Dependencies / sequencing notes

- Profile birth expands the corpus (M1 gets a Classroom channel)
  whenever Noah + Sam do the supervised session — independent of
  this plan's start.
- Building the second estate MCP tests the deployment pattern
  again; whatever repeats becomes the `mcp-kit` extraction
  candidate (deferred in docs/PACKAGES.md).
