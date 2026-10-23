# Classroom corpus — pulling all docs local for agentic use

## Goal

A complete local copy of Sam's Google Classroom documents —
Slides, Docs, Sheets, PDFs, attachments, and linked materials —
so agents can search, read, and reason over the actual content,
not just titles and due dates.

## Invariants

- **Every item keeps its original Classroom deep link.** The corpus
  is a working copy for agents; student-facing references always
  cite the Classroom original (persona rule).
- Store, per item: the original file (or best available export),
  extracted text, and metadata — course, source (assignment /
  material post / resource link), Classroom timestamps, deep link.
- The corpus lives on hal9k, beside the index built from it.

## Scope

ALL docs (Noah, 2026-10-23): class materials and the files attached
to assignments across Sam's classes, plus the resources the adapter
already surfaces as links. If it teaches or assigns, it's in.

## Acquisition channels (in preference order)

1. **Browser-driven via Sam's Classroom profile** on hal9k (the
   adapter's browser driver; profile birth is the supervised step
   already pending). The profile session can walk Stream/Materials
   per class and download attachments/exports. Proven channel —
   the login test passed cleanly; challenges escalate, never fight.
2. **Drive API** with Sam's OAuth — almost certainly district-gated
   like the Classroom API was (Stage 0 finding). Worth one cheap
   test; do not design around it.
3. Public/teacher-published links where they exist (the agenda
   decks already work this way).

## Sync shape

- Initial full pull after profile birth; incremental thereafter
  (drive by modified time / stream position; the orchestrator
  schedules it, Hermes is a candidate runtime for the batch job).
- Changed doc → new version stored, index re-embeds that item.
- Removed doc → tombstone (drop from index; keep metadata so old
  pointers fail honestly rather than silently).

## Extraction

- Slides: the adapter's SVG-accessibility extraction already reads
  published decks; profile-downloaded decks export to text/PDF.
- Docs/Sheets: export to plain text / CSV. PDFs: text layer first,
  OCR only if a doc proves to be image-only (decide when seen).

## Open questions

- Storage layout + retention (school-year rollover? keep prior
  years for Aiden/Owen reuse of the pipeline, with Sam's corpus
  itself archived at year end).
- Whether Aiden/Owen corpora follow the same pipeline later
  (generalization was an adapter design goal from the start).
