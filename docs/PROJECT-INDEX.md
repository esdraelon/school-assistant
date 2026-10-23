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

## Milestones

- **M0 — Skeleton + standards.** Repo scaffold per the coding
  standard (verify.sh gates from day one); the `school-core`
  question decided by evidence: consume resource types via the
  adapter's store/gateway, or extract — record the call in
  DECISIONS.md.
- **M1 — Corpus sync (reachable sources).** Harvest the adapter's
  current resources; store originals + metadata; idempotent
  re-runs; sync watermark + tombstones.
- **M2 — Extraction.** Text out of every harvested format;
  extraction failures are per-item and visible, never silent.
- **M3 — Chunk + embed (ONNX).** Strategy registry in place;
  incremental re-embed on changed items only.
- **M4 — Index MCP + deploy.** Gateway route (own token, path
  scoped), three query tools, deep links in every result; deploy
  per the runbook; smoke through the public route.
- **M5 — RAG benchmark.** ~30 hand-labeled queries from the real
  corpus; Bedrock vs ONNX pointer/passage accuracy; results in
  docs/RAG-EVALUATION.md; the stack decision recorded in
  DECISIONS.md (Noah's call, evidence attached).
- **M6 — Tutor-facing shapes.** The exact query/response shapes the
  student persona needs (pointer composition inputs, worked-example
  retrieval), verified against personas/student.md.

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
