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
