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
