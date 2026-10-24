# Coding standard

Distilled from the school-info-adapter build (M0–M7.5), which is the
reference implementation. Applies to all school-* packages.

## Language and shape

- **Python 3** (3.12 in production images), src layout, one package
  per area with clean internal modules.
- **No file over 500 LOC.** Split by responsibility before it grows.
- **Every module names its design pattern** in its docstring
  (Strategy, Adapter, Composition Root, …) — the pattern is part of
  the module's contract.
- **Frozen dataclasses, keyword-only construction.** No long
  positional constructors.
- **Composition and higher-order functions over repetition.**
  Pipelines are composed, not copy-pasted.

## Selection and absence

- **No if/else chains for config-driven selection.** Use dependency
  injection + strategy registries: a Mapping from key → strategy,
  injected at the composition root. (An embedding provider, a deck
  parser, a source driver — all registries.)
- **Absence is explicit:** `Lookup[T]` (`Found` / `Missing`) at
  module boundaries in the adapter. Not used as a general parameter
  or return-type decoration — where a value is simply optional,
  `X | None` with mypy strict says so.

## Testing and gates

- **≥90% unit coverage** (total and per key module), enforced in
  `scripts/verify.sh` alongside mypy strict and the LOC cap.
- **System tests mock only external dependencies** — real registry,
  real merge, real store, real HTTP; the network edge is the only
  substitution. If a suite can't run in an environment, it skips
  with the precise reason recorded, and a fallback carries the
  assertions until the real run happens (and is then reconciled —
  fixtures must agree with ground truth; see the M6/M7.2 lesson).
- **Verify, don't trust:** whoever dispatches a milestone re-runs
  the gates independently before accepting it.

## Pagination & coverage (standing rule, from the 2026-10-23 incident)

On 2026-10-23 the school adapter's `get_drivers` returned exactly
200 items in one default call, oldest-first, and a Science Unit
Assessment due 2026-10-23 was invisible until the query was
windowed by due date. The 2026-10-24 live audit confirmed the
shape on both collection tools: default limit 200, silent
truncation (bare list, no total, no `has_more`), no cursor/offset
on either tool. A capped list presented as complete is a
data-corruption bug — the index harvester did exactly that damage
(truncated corpus, plus false tombstones for live items that
drifted past the title-200 boundary). Rules:

- **Every collection tool documents its contract:** default limit,
  ordering, how a consumer pages to exhaustion (cursor/offset, or
  the partition params where no cursor exists), and what
  truncation looks like (silent vs signaled). The adapter's
  contract lives in its `deploy/README.md`; a tool without this
  section is incomplete.
- **Every consumer pages to exhaustion.** Where a cursor exists,
  loop to it. Where none exists, partition (time windows, course,
  kind) so every single call provably returns fewer than the
  limit, deduplicate the union, and cross-check it against an
  independent leg. A page returning exactly the limit is treated
  as possibly-truncated, never as complete.
- **Every coverage claim proves total + bounds in tests:** total
  count and min/max bounds (due range for drivers, title range or
  partition-union count for resources) against a seeded store
  LARGER than the default limit. A test whose fixture fits under
  the cap proves nothing about paging and is a gate failure.
- **A single default-limit call in a coverage path is a gate
  failure** — in code review, in `scripts/verify.sh` checklists,
  and in soak/parallel-run comparisons. Narrow-window calls are
  not exempt: each window needs its own < limit assertion, and
  windowed queries must account for what the window excludes
  (e.g. the adapter's due windows exclude undated items, which
  need their own unwindowed leg).
- **Degraded inputs degrade visibly.** A consumer that cannot
  prove exhaustion says so (coverage banner naming the bound),
  never silently serves the capped set as the whole world.
  Tombstoning, deleting, or overwriting on the basis of a capped
  page is forbidden: destructive folds require the exhaustive set.

### Open item — agent-at-ingestion validation (NOT scoped work)

From `docs/REFINEMENT-NOTES.md` (Noah, 2026-10-24; consideration
only — waits for Noah before it becomes a milestone): an agent
could be brought to bear during INGESTION to validate soft
formatting drift that deterministic parsers pass silently (random
HTML fragments leaking into extracted text, deck-template drift,
malformed structure). Candidate placements, all at ingestion and
never at read time (runtime personas consume stores as ground
truth, so verdicts must be settled before the write):

- a `doc_extract` post-pass flagging markup artifacts in
  extracted text;
- a `corpus_sync` change gate validating a new/changed item
  against the previous version's shape before overwrite,
  quarantining shape drift instead of overwriting;
- validator review of low-confidence email extractions before
  they reach the events store (calendar collectors; same flag,
  not silent-feed, discipline as `docs/CALENDAR-EVENTS.md`).

Open questions (unsettled): sampling vs full pass; quarantine-vs-
flag thresholds; whether the validator only flags/quarantines or
also writes corrections (flagging is the safer default — a
verdict recorded as provenance, like freshness, on the stored
item). This paragraph is the plan amendment the refinement note
asks for. It authorizes no work: if a milestone plan would absorb
it, that plan records it as a follow-up instead.

## Process

- **Plan before building; work checklist before code.**
- **One agent per milestone**, tight context, discrete commits on
  **stacked branches**. Main fast-forwards to the verified tip.
- **Reconcile against ground truth early** — real cache schemas,
  real page structures, real fixtures. Assumed shapes are the
  most expensive bug class this project has seen.

## MCP deployment standard (deploy-proven, 2026-10-22)

- Serve the MCP at the **app root** (`streamable_http_path="/"`).
  A sub-path mount canonicalizes with a 307 redirect, and redirects
  break behind the gateway.
- **Allowlist the public host** in the SDK's DNS-rebinding
  protection (`TransportSecuritySettings`): the default derives
  from the bind address and 421s the gateway's Host.
- **uvicorn proxy headers on** (`proxy_headers=True`,
  `forwarded_allow_ips="*"` in the serve path); nginx pins
  `X-Forwarded-Proto https` (the vhost is the TLS terminator).
- **Gateway wiring mirrors the sibling MCPs:** nginx
  `auth_request /_auth`, `proxy_pass` with URI part stripping the
  route prefix, `Host $http_host`, long read timeout, buffering off.
- **One token per MCP**, registered in the gateway token registry
  scoped to exactly its path; the app re-checks the same token.
  Clients present that token alone.
- Container conventions: non-root user, localhost-only port,
  persistent state on a mounted volume owned by the container uid,
  config via environment, a systemd timer (not in-container cron)
  for scheduled passes.
- Herald the deploy lessons in the runbook when they change:
  the adapter's deploy/README.md is the template.
