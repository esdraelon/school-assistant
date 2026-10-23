# RAG evaluation — embeddings + vector store for the corpus index

**Status: open decision (Noah's).** This doc frames the options; it
does not settle them.

## What the index must do

- Passage search over the extracted text of the full Classroom
  corpus (notes, slides, docs, PDFs): "material about photosynthesis,"
  "worked examples of two-step equations."
- Topic → resource mapping: which resources cover topic T, returned
  with their **original Classroom deep links** (pointer answers).
- Freshness: re-embed changed docs incrementally; the corpus sync
  drives this.

## Scale (why this is a small problem)

One student's school year: hundreds of documents, low thousands of
chunks. Any candidate below handles this trivially; the decision
rides on cost, privacy, ARM performance, and ops weight — not on
scale-out.

## Criteria

1. **Privacy** — student schoolwork; keeping content on ululuth is
   preferred, sending text to a cloud API is a deliberate tradeoff.
2. **Cost** — standing household preference: no per-call API spend
   for always-on personal infra (ululuth + Hetzner only).
3. **hal9k fit** — ARM64, shared with the rest of the estate; heavy
   resident services are disfavored (same reasoning that kept the
   adapter off the JVM).
4. **Retrieval quality** — good enough that pointer answers land on
   the right resource nearly every time; measurable on a small
   hand-labeled sample from the real corpus.

## Embeddings options

- **Amazon Bedrock** (Titan Embed / Cohere Embed): managed quality,
  zero local compute. Costs per call, content leaves ululuth to AWS,
  needs AWS credentials wired in. The quality benchmark candidate.
- **Local ONNX** (e.g. bge-small / bge-m3 via onnxruntime): free,
  fully private, CPU inference at this scale is fine (batch-embed on
  sync, not on the query path's critical path). Model management and
  ARM wheels are on us. Fits the estate's grain.
- Local PyTorch / sentence-transformers: same models, heavier
  footprint than ONNX for the same result — no reason to prefer it.

## Vector store options

- **sqlite-vec**: single file beside the corpus DB, estate-consistent
  (the adapter store is SQLite), brute-force at this scale is
  milliseconds. Least new infrastructure by far.
- **pgvector**: needs a Postgres instance; more moving parts than
  this scale justifies.
- **Qdrant**: a real vector DB with an ARM image; clean API, but a
  new resident service to babysit for thousands of vectors.
- **LanceDB / Chroma**: embedded options; viable, but no advantage
  over sqlite-vec inside this estate.

## Recommendation sketch (not the decision)

ONNX embeddings (bge family) + sqlite-vec for v1: zero marginal
cost, private, no new services. Before committing, run Bedrock and
ONNX over the same ~30-query labeled sample from the real corpus
and compare pointer accuracy — if Bedrock is decisively better,
that's a real input to the decision; if it's a wash, local wins on
criteria 1–3.

## Out of scope here

Generation is the agents' job (Muse personas); the index MCP only
retrieves. A local generation model (e.g. via Hermes tasks) is a
separate question, raised only if a sub-agent ever needs one.

## Benchmark results (2026-10-23)

Ran on hal9k against the live corpus: 38 searchable items / 371
chunks (of 99 harvested — the rest failed harvest or carry no
embeddable text yet), labeled set v1.0 (33 tutor-style queries
labeled from a same-day corpus census: 29 scored, 4 absent-topic
probes). Every number below re-derives from the records in
`~/school-index-data/benchmark.db` via
`python -m school_index.benchmark report`; per-query raw rankings
are stored there, not summarized away.

| Arm | Status | Res hit@1 | Res hit@3 | Res MRR | Pas hit@1 | Pas hit@3 | Pas MRR | Top-chunk | Separation (res/pas) | Latency |
|---|---|---|---|---|---|---|---|---|---|---|
| onnx-production (bge-small-en-v1.5 + sqlite-vec, production chunks) | completed | 0.83 | **0.93** | 0.894 | 0.97 | **1.00** | 0.977 | 0.97 | +0.099 / +0.104 | ~30 ms mean (24–45) per query, both searches |
| bedrock-titan (amazon.titan-embed-text-v2:0) | **BLOCKED** | — | — | — | — | — | — | — | — | — |
| onnx-chunk600 (600-char chunking variant) | not run | — | — | — | — | — | — | — | — | — |

**Bedrock arm — blocked, with the exact reason:** no AWS
credentials exist on hal9k (checked 2026-10-23: no `~/.aws`, no
`AWS_*` environment), and the probe records it verbatim —
`NoCredentialsError: Unable to locate credentials`. The provider
is built and unit-tested (boto3 mocked at the edge), the boto3
extra is installed in the project venv, and the region is pinned
(us-east-1): the arm is exactly one credentials file away from
running. Nothing was hunted for or improvised.

**Chunking arm — not run, by the plan's own rule:** the variant
exists to test whether pointer accuracy is chunk-bound (trigger:
production resources hit@3 < 0.7). Production scored 0.93, and the
two misses below are not chunk-size failures, so the arm was
skipped and the call recorded here rather than run for ceremony.

**The two resources misses (of 29):**
- *Colonial trade simulation* — expected deck ranked 4th in the
  item aggregate, behind the Team Weekly Agenda, the colonization
  reading, and the CSI storefront page. Its best chunk still hit
  (passages found it): the breadth-bonus aggregate rewards the
  63-chunk omnibus over a 3-chunk deck. An aggregation artifact,
  not an embedding or chunking failure.
- *Science units this year* — the only total pointer miss (rank 6,
  top chunk also wrong). All three schedule documents (team
  agenda, tutorial schedule, ELA daily slides) outranked the
  science year-at-a-glance, a table-dense spreadsheet whose chunks
  embed weakly against a prose question.

**Omnibus behavior, measured:** the schedule documents do not
steal topical wins — the agenda places 2nd on the Pythagorean and
Boston Tea Party queries but loses by ~0.18–0.20 of cosine — yet
they are the corpus's universal fallback: the Team Weekly Agenda
is the top hit on **all four absent-topic queries**, at
0.57–0.66. Absent-topic top scores (mean 0.626) overlap genuine
hit scores, and separation, while positive, is thin (+0.10). A
raw-score threshold alone cannot tell "found it" from "invented
relevance" — that is a design input to the tutor (M6), not a
reason to change embedders.

**Recommendation (the decision stays Noah's):** keep ONNX
bge-small + sqlite-vec — the incumbent production stack. At
hit@3 0.93 / passages 1.00, 30 ms on ARM, zero marginal cost,
fully private, Bedrock's maximum possible gain is the remaining
0.07, bought with a standing AWS dependency and per-call spend —
against criteria 1–3 above, that trade fails. Revisit Bedrock
only if AWS credentials land on hal9k for another reason, or the
post-Classroom corpus (much larger) degrades local quality. The
lever the benchmark actually exposes is ranking/calibration:
chunk-level results beat the item aggregate everywhere, so if
pointer quality needs work, it is the resources aggregation and
omnibus handling that want it — not the embedding model.
