# Skill: development & orchestration in the school-* estate

For any agent (Haliffer, Sam's agent, Hermes tasks, milestone
sub-agents) doing build or operations work in this system.

## Before anything

1. Read `CONTEXT.md` (repo root) and the relevant persona/doc files.
2. Check ground truth before designing against it: real schemas,
   real pages, real configs — read them, don't remember them.
3. Plan before building: a written plan + a work checklist exist
   before the first line of code. Scope is the plan; anything the
   plan didn't authorize waits for Noah.

## Building (per package)

1. Follow `docs/CODING-STANDARD.md` — it is the distilled adapter
   playbook (DI/strategy registries, Lookup at boundaries, 500-LOC
   cap, named patterns, ≥90% coverage, mypy strict).
2. Work in **milestones**, one agent per milestone, tight context:
   the milestone brief names its scope, its gates, and what it must
   NOT touch.
3. **Discrete commits on a stacked branch**, one concern per commit.
4. The dispatcher **re-runs `scripts/verify.sh` independently**
   before accepting a milestone. An agent's self-report is input,
   not verification.
5. Reconcile fixtures against reality the moment ground truth is
   reachable (run the suite where the real dependency exists —
   e.g. Chromium tests on hal9k, not in a sandbox that can't).
6. Push every milestone branch to the ululuth remote; main
   fast-forwards to verified tips only.

## Deploying an MCP

Follow the adapter's `deploy/README.md` runbook shape and the
coding standard's deployment section. Non-negotiables:

- Human go before production changes; enumerate the checkpoints
  and confirm each green before advancing (container up → scrape
  → health → gateway smoke through the public route with the real
  protocol → existing routes undisturbed).
- One token per MCP, path-scoped in the gateway registry.
- Rollback is a config/route change, never a data migration;
  the thing being replaced stays up until the new one soaks.

## Orchestrating (runtime agent work)

- Sub-agents get **tight context**: persona file, config, the
  specific tools for the task — nothing else.
- State lives on hal9k (stores, logs, watermarks); agents are
  replaceable workers. A run that changes state records what it
  did where the next run can see it.
- Degraded inputs degrade outputs **visibly** (staleness banner,
  named failed source). Never present stale data as fresh, never
  silently drop a source.
- Escalate, don't improvise, at hard boundaries: auth challenges,
  missing credentials, production changes outside the brief.

## Communication

- Report what was verified, what the evidence is, and what remains
  open — in that order, briefly. Own misses in one plain line and
  fix them. Closed threads stay closed.
