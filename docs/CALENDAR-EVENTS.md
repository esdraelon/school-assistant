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

## Coordination

This is inherently multi-agent: calendar owners, email readers, and
the orchestrator each hold part of the picture. The orchestration
agent consumes normalized events (title, start/end, source,
confidence, source link) from whichever agentic source produces
them; it does not scrape calendars itself. Source agents keep their
own freshness; the orchestrator merges and dedups (same
precedence/freshness discipline as the school adapter's merge).

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
