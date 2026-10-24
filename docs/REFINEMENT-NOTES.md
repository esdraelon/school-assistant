# Refinement notes

Open considerations, not scoped work. Anything here waits for Noah
before it becomes a milestone.

## Agent-at-ingestion validation for soft formatting drift

**Noted 2026-10-24 (Noah).** Ingested material drifts softly:
random HTML fragments leaking into extracted text, deck-template
drift, malformed structure that deterministic parsers pass through
silently. The estate's expensive bug class is assumed shapes; a
parser cannot judge "does this extraction still look like the
source's intent?" — an agent at the intake boundary can.

Candidate placements (all at ingestion, never at read time —
runtime personas consume stores as ground truth, so validation
verdicts must be settled before the write):

- **doc_extract post-pass:** flag items whose extracted text
  carries markup artifacts (HTML tags, SVG leftovers, encoding
  mojibake) or whose structure deviates from the known deck
  template.
- **corpus_sync change gate:** validate a new/changed item
  against the previous version's shape before it replaces the
  stored one; shape drift quarantines instead of overwriting.
- **Calendar collectors:** email-derived events already carry
  extraction confidence; a validator reviews low-confidence
  extractions before they reach the events store (same flag, not
  silent-feed, discipline as CALENDAR-EVENTS.md).

Open design questions (unsettled, not decided):

- Sampling vs full pass — cost and latency against coverage.
- Thresholds: what drift quarantines an item vs merely flags it.
- Does the validator write corrections, or only flag + quarantine
  for review? (Correction-writing reopens the "interpretation
  happens once" question — a verdict recorded as provenance is
  the safer default.)
- Validation verdict as a record property (like freshness and
  provenance) on every stored item.

Status: consideration only. Fold into Phase 0 plan amendments /
calendar collector design as an open item, not as scope for any
in-flight milestone.
