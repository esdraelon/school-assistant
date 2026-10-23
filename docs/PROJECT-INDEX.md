# First project: school-index v1

**Decision (2026-10-23):** the index is the first build. Tutor and
study block on it; Guardian does not (Guardian alerts ride the
digest v2 cutover already soaking, then become the first
orchestrator package).

**Status: COMPLETE 2026-10-23 (M0–M6).** Built without origin
repos per Noah (ululuth is the origin). Closeout evidence under
Acceptance; recorded issues follow it.

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
- [x] Repo scaffold: src layout, pyproject, `scripts/verify.sh`
      (tests, ≥90% coverage, mypy strict, 500-LOC cap) green on an
      empty package.
- [x] Verify `get_resources` against the live adapter (pagination,
      counts, deep-link fields) — evidence for the boundary call.
- [x] Decide: consume resource types via the adapter's MCP contract
      or extract `school-core`. Record in DECISIONS.md.
- [x] **Gate:** verify.sh green; decisions recorded.

### M1 — Corpus sync (reachable sources)
- [x] `corpus_items` + `sync_state` schemas; store module.
- [x] Harvester over the adapter's resources: download/export,
      content hash, idempotent re-runs (second run = no changes).
- [x] Tombstones for vanished resources; per-item failure records.
- [x] Fixture corpus from real resource shapes; system test mocks
      only the HTTP edge.
- [x] **Gate:** full harvest of Sam's current resources completes;
      re-run is a no-op; coverage ≥90%.

### M2 — Extraction
- [x] Extractor registry (format → strategy); Slides (SVG-a11y),
      Docs/Sheets export, PDF text layer, HTML/text passthrough.
- [x] `item_text` populated per item; per-item failure containment
      with the failure stored, not logged-and-lost.
- [x] **Gate:** every ok corpus item has text or a failure record;
      no silent skips in the run report.

### M3 — Chunk + embed (ONNX)
- [x] Structure-aware chunker; `chunks` schema.
- [x] Embedding strategy registry; ONNX (bge) arm working on
      hal9k (ARM); sqlite-vec store; `embeddings` keyed by chunk +
      model_id.
- [x] Incremental: only changed chunks re-embed; model change =
      explicit full re-embed op.
- [x] **Gate:** index builds from the real corpus; a changed item
      re-embeds only its own chunks.

### M4 — Index MCP + deploy
- [x] Three tools per the design (`search_passages`,
      `resources_for_topic`, `worked_examples`); every result
      carries title + course + deep link + provenance.
- [x] Deploy per the runbook: container, `/mcp/index/` route, own
      token (`index-mcp`, path-scoped), volume ownership, timer.
- [x] Smoke through the public gateway route (real protocol);
      sibling routes undisturbed.
- [x] **Gate:** live queries return traceable results; negative
      auth checks pass; deploy notes land in deploy/README.md.

### M5 — RAG benchmark
- [x] ~30-query labeled set from the real corpus (topics, expected
      resources/passages), versioned as data.
- [x] Arms: ONNX+sqlite-vec vs Bedrock (same chunking); optional
      chunking arm if accuracy looks chunk-bound.
- [x] Results stored as `benchmark_runs` records; summary into
      docs/RAG-EVALUATION.md; stack decision recorded in
      DECISIONS.md (Noah's call).
- [x] **Gate:** benchmark reproducible from the stored records.

### M6 — Tutor-facing shapes
- [x] Walk personas/student.md flows (pointer answers, similar-
      problem composition inputs, study-plan resource lists)
      against the live MCP; file gaps as issues, fix what's cheap.
- [x] Response shapes frozen as the tutor contract (fixtures).
- [x] **Gate:** every student-persona query pattern returns what
      the persona needs, deep links intact.

## Acceptance

Gates green per the coding standard; index answers from the public
gateway route; every result traceable to an original via its deep
link; benchmark done and the RAG decision made with data.

### Evidence (closeout, 2026-10-23)

- **Gates:** verify.sh green on the dev machine AND hal9k at the
  M6 tip (273 unit tests, 98% coverage, mypy strict, LOC cap,
  7 system tests).
- **Public route:** the M6 redeploy re-ran the runbook
  checkpoints — container up → local answer → gateway smoke (all
  three tools, envelope shapes) → negative checks (no token 401,
  cross-token 403 both ways) → siblings undisturbed.
- **Traceability:** the M6 persona walkthrough (knowledge
  pointer, similar-problem inputs, study-plan list,
  why-is-X-wrong) ran through the public gateway route; every
  top result carried its original deep link.
- **Benchmark:** M5 + M6 runs stored in benchmark.db (reproduce
  with `python -m school_index.benchmark report`). After M6's
  banded aggregation: resources hit@1 0.97 / hit@3 1.00 / MRR
  0.977 (M5: 0.83 / 0.93 / 0.894); passages unchanged (hit@3
  1.00, MRR 0.977) — no resources hit@3 misses remain.
  `match_quality` validated on the labeled set (live service and
  stored records agree exactly): absent probes 4/4 weak on both
  tools; scored tops strong 28/29 resources / 27/29 passages.
- **RAG decision:** the evidence and the recommendation (keep
  ONNX + sqlite-vec) are in docs/RAG-EVALUATION.md. The decision
  itself is Noah's and stays open by design — "made with data"
  is satisfied by the evidence package, not by pre-empting it.

### Issues recorded at closeout (not fixed — out of M6 scope)

- **Why-is-X-wrong phrasings under-score.** A misconception-style
  query ("why is the hypotenuse not just a+b") embeds worse than
  the topic-style queries the calibration derives from: the
  needed passage ranked #2 but the set calibrated "weak". The
  mitigation is tutor-side: reformulate to the underlying concept
  (the walkthrough's concept query scores "strong") and use the
  passages as explanation support. A misconception-style labeled
  set is the input if this is ever calibrated index-side.
- **Title ambiguity in the corpus.** Some items carry an empty
  title and several share a title ("KEY"); pointers stay
  unambiguous by deep link, but the tutor should present course +
  link, not title alone. A display-title derivation belongs to a
  future corpus_sync pass.
- **worked_examples calibration is inherited.** Its reformulated
  query was never separately benchmarked (stated in the contract
  doc); v1 returns passages that read as worked material, not
  verified examples — the walkthrough's #2 hit was a perimeter
  problem from the Distance deck: relevant, not on-topic.

## Dependencies / sequencing notes

- Profile birth expands the corpus (M1 gets a Classroom channel)
  whenever Noah + Sam do the supervised session — independent of
  this plan's start.
- Building the second estate MCP tests the deployment pattern
  again; whatever repeats becomes the `mcp-kit` extraction
  candidate (deferred in docs/PACKAGES.md).
