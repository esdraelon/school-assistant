# Calendar + events — sources, coordination, and the feed

## Goal

One trustworthy picture of events and extracurricular dates/times —
band, choir, school events — feeding (a) the Guardian calendar feed
and (b) lead-time event notifications alongside the due-date watch.

## Sources (inventory first)

Events live in **various calendars and emails** (Noah, 2026-10-23).
Known/expected sources, to be inventoried before any build:

- **The configured 8th grade calendar** — carries some of it.
  Inventory: which calendar is this, who owns it, what event types
  are already on it, what format/access (ICS? Google Calendar?).
- Other calendars: band/choir ensemble calendars, district/school
  calendars, family calendars.
- **Email-derived events:** director emails, school newsletters,
  schedule-change notices. These arrive as prose; extracting an
  event (what/when/where) is an agent task, and dedup against the
  calendar entries is required so a concert isn't double-counted.

## Coordination — shared store; dispatch mechanism deferred

**Status (2026-10-23):** the events store and its normalized
schema are settled — they follow from the data discipline
(docs/ARCHITECTURE.md): personas bind to stored records, not to
how the records got there. How collectors are scheduled and
dispatched is **deferred** (Noah: cross that bridge later), and
the store is what makes deferring safe. The leading candidate:

The backbone is a **blackboard**: a shared events store owned by
`school-calendar`, on hal9k, that every participant reads/writes
through a contract. Agents do not message each other to coordinate;
they publish and consume state. This matches the estate's grain
(MCPs as shared resources, hal9k as system of record, agents
replaceable).

**Collectors.** Each source has exactly one owning collector — a
scheduled, tightly-scoped task or sub-agent:

- **Calendar collectors** are mostly not agents at all: ICS/Google
  Calendar reads are tool calls on a schedule (the estate's
  connectors already read both mail and calendars).
- **The email collector is an agent task** — prose → events is
  extraction, not parsing. It reads the school/band mailboxes,
  extracts candidate events, and publishes them with a link to the
  source message. (Scope is events/schedules only; absence notices
  are Noah's to read and are never surfaced — standing boundary.)

**The contract.** Collectors publish *normalized events* to the
store via the calendar MCP's ingest tools:

- title, start/end (with timezone), location;
- provenance: source kind (email | calendar), source id/link,
  collector identity, extraction confidence;
- per-source health: last successful collection, recorded in the
  store exactly like the adapter's per-source scrape health — a
  dead collector reads as a stale source, not as "no events."

**Merge.** The calendar package owns merge/dedup with the
adapter's discipline: calendar entries outrank email mentions of
the same event; explicit updates supersede; low-confidence email
extractions are flagged, not silently fed. Consumers (Guardian
feed, orchestrator) read only the merged store.

**The exception path.** Direct agent-to-agent requests exist only
for live questions the store can't answer ("search the mail for
anything about Saturday's rehearsal") — routed as a task to the
owning agent, answered out of band, never a standing protocol.
If a question recurs, it becomes a collector, not a conversation.

## Outputs

- **ICS feed** on haliffer.ululuth.com (token-gated, same pattern
  as the gateway): due dates that matter (quizzes/tests as all-day
  entries) + events/extracurriculars. Subscribable from the family
  calendar apps.
- **Guardian notifications** for events at the global lead time,
  same engine as due dates.

## Boundaries

- Due dates are drivers (school MCP), not events — they project
  onto the feed; they don't move into a calendar store.
- An email that *mentions* a date is not an event until an agent
  extracts it with a source link; low-confidence extractions are
  flagged, not silently fed.
