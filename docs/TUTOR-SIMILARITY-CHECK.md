# Design: tutor similarity check

Status: **design doc, 2026-10-24.** No code. This feeds a later tutor
build; it does not authorize building the tutor, and it does not
settle any open decision (see §7 — the mechanism decision stays
Noah's, recorded in docs/DECISIONS.md when he makes it).

## Context it composes with

- **Persona** (personas/student.md): the tutor never answers the
  assigned problem. For math it composes a *similar* problem — same
  structure and skill, different numbers/context — walks through
  that one fully, then hands the assigned problem back to Sam.
  For knowledge questions it does not answer from its own head: it
  returns a *pointer* to the notes/slides/Classroom resource, with
  the original Classroom deep link.
- **Tutor contract v1** (school-index repo, docs/TUTOR-CONTRACT.md,
  frozen 2026-10-23): the index MCP's three tools —
  `search_passages`, `resources_for_topic`, `worked_examples` —
  each return an envelope with `match_quality` (`strong`/`weak`)
  and every result carrying a deep link + provenance. The
  similarity check **composes with** those tools; it does not
  replace them, and it does not change their shapes.
- **Estate rules** (docs/CODING-STANDARD.md, development-orchestration
  skill): MCPs stay single-purpose and policy lives in the
  orchestration layer — so this check lives in the future `tutor`
  package, **not** in the index MCP. The index stays retrieval-only.
  Degraded inputs degrade outputs **visibly**: never silently.
  Operating evidence is a schema-validated record in a store on
  hal9k — a check verdict is a record, not a chat aside.
- **Scale** (docs/RAG-EVALUATION.md): one student's school year —
  hundreds of documents, low thousands of chunks. Embedding a
  candidate problem at check time is milliseconds on the estate's
  ONNX bge-small stack. Nothing in this design needs scale-out.
- **A calibration precedent to copy, not a number to reuse:**
  `match_quality` was derived from the measured distributions of
  one 33-query benchmark run (absent-topic tops overlap genuine
  weak hits on raw score alone; margin and breadth signals separate
  them), and its constants are documented as corpus- and
  model-specific, to be re-derived when the corpus grows or the
  model changes. Every threshold in this design follows the same
  rule: **measured from distributions, never guessed** (§6).

## 1. The requirement, restated precisely

Let **A** = the assigned problem, **C** = the generated candidate
"similar" problem. The check answers one question, before C reaches
Sam: *is C the assigned problem restated?*

### 1a. Math (numeric / procedural problems)

C is acceptable iff **all** of these hold:

1. **Same skill, same structure.** C exercises the same concept and
   the same solution-step shape as A (e.g. two-step linear equation,
   Pythagorean hypotenuse solve, percent-of-a-quantity). Same
   template with different numbers is not a failure — it is the
   *desired* outcome. The persona says "same structure and skill,
   different numbers/context", and the check must not punish the
   first half of that sentence.
2. **Different instance.** C's numeric instance differs from A's in
   a way that is not cosmetic. Cosmetic changes **count as A** and
   fail the check, including:
   - identical numbers, reworded;
   - the same numbers with entities renamed (apples → oranges,
     Sam → Alex), context swapped, or sentences reordered;
   - the same numbers under unit conversion (3 m → 300 cm) or a
     different variable name (x → y);
   - the same equation with terms commuted (2 + 3x = 8 → 3x + 2 = 8)
     or otherwise reordered without changing the instance;
   - the same numbers with one trivially-derived value substituted
     (e.g. a side given as 6 replaced by "half of 12" where the
     problem hands Sam the derivation for free).
3. **Solving C does not hand Sam A's answer or A's worked steps.**
   C's walkthrough must not be A's walkthrough with find-and-replace.
   If the steps, intermediate values, or final answer of C coincide
   with A's beyond what the shared template forces, that is
   evidence of restatement, not of similarity (see the
   answer-collision signal in §2 and the open question in §7 about
   whether answer coincidence alone fails a candidate).

What the check is **not**: it is not a difficulty matcher (C may be
somewhat easier or harder than A — that is generation quality, not
identity), not a correctness prover for C's walkthrough (the
walkthrough is checked by the tutor's own generation discipline),
and not a plagiarism detector over the corpus (C is generated, not
retrieved — though §4's fallback *retrieves* a worked example, and
that retrieval is governed by the contract, not by this check).

### 1b. Knowledge questions (where the tutor points, not composes)

For knowledge questions the check **does not apply to the pointer
itself** — and that distinction matters:

- Pointing Sam at the resource that covers the topic *is* the pedagogy.
  The pointer is supposed to be the source material, cited by its
  original Classroom deep link. A pointer that "matches" the question
  is a success, not a leak. Gating pointers on dissimilarity would
  invert the persona.
- The check **does** apply to any text the tutor *generates around*
  the pointer: a paraphrase/summary of the resource, a reformulated
  practice question in study mode, or a why-is-X-wrong explanation.
  If that generated text reproduces the answer to a graded
  knowledge question (an essay prompt, a short-answer response, a
  definition asked verbatim), it fails for the same reason a
  restated math problem fails — the tutor did the graded work.
  In practice this means: the knowledge path's gate is a
  *generated-text* gate (verbatim/near-verbatim overlap against the
  source passage, §2), and the default safe behavior is to point
  with a *locator* ("slides 4–6, the definition box") rather than a
  summary that contains the answer.
- Study-mode practice questions sit between the two cases: a
  generated practice question about a topic is gated like a math
  problem against the known assigned/test item when one is known,
  and against the just-solved practice item otherwise (to avoid
  serving the same question twice under new wording). When no
  assigned item is known, the gate can only check
  self-repetition, and the verdict must say so (§4).

## 2. Mechanism survey

### (i) Embedding-distance threshold against the assigned problem text

**How:** embed A and C with the estate's ONNX bge-small embeddings
(the same model behind the index; no new service, no API spend,
milliseconds at this scale) and compare cosine similarity. A
candidate "too close" to A fails.

**What it is good for:** a coarse, cheap gate. It reliably catches
verbatim copies and close paraphrases, and it catches generation
drift — a candidate that is *far* from A is off-topic, which is a
generation failure even though it is not a restatement failure
(the two failure directions need different handling, §4).

**Why it cannot be the primary mechanism:** embeddings measure
*topical* similarity, not *instance* identity — and for this
requirement those two are nearly orthogonal. A same-template,
different-numbers problem (the desired outcome) and a
same-template, same-numbers restatement (the forbidden outcome)
differ only in a few numeric tokens, which bge-small weights
lightly; both embed at essentially the same distance from A. The
benchmark already taught this lesson in another form: absent-topic
tops overlap genuine weak hits on raw score alone (mean 0.626 vs
the genuine distribution, separation only +0.10), which is why
`match_quality` needed margin and breadth signals on top of raw
score. A single cosine threshold would inherit exactly that
overlap, in a harder place: the two classes the check must separate
(desired-similar vs forbidden-restated) are *closer* to each other
than "found" vs "absent" were.

**Threshold derivation (the non-negotiable part):** if an embedding
signal is used at all, its threshold must be derived the way
`match_quality` was — from measured distributions over labeled
pairs (§6's fixture set: identical, cosmetic-restatement,
same-template-different-numbers, different-problem-same-topic,
different-topic), plotting the cosine distribution per class and
placing any gate in an observed gap, or reporting that no gap
exists. A threshold picked by intuition (0.9, 0.95, "high") is not
a design; it is a guess with a gate attached. Constants are
model- and corpus-specific and must be re-derived when either
changes — post-Classroom corpus expansion will move them, as
CONTEXT.md already warns for the `match_quality` constants.

**Weakening under paraphrase input:** when the tutor only has Sam's
paraphrase of A (§4), the embedding reference itself shifts. A
paraphrase that drops numbers or context embeds differently from
the original, so both the "too close" and "too far" readings get
noisier — the gate's discriminating power falls exactly when the
tutor is least sure what A is.

### (ii) Structural / template comparison

**How:** reduce A and C to a *template signature* — the solution-step
shape with numeric literals and entity names abstracted out — and
compare signatures plus the abstracted-out slots separately:

- same signature + same slot values → restatement → **fail**;
- same signature + different slot values → desired similar → **pass**;
- different signature → a different problem (acceptable only if it
  still exercises the target skill; otherwise a generation-quality
  miss, not an identity failure — the tutor should prefer
  same-signature candidates and treat a different-signature
  candidate as a weaker pass, recorded as such).

For a word problem the signature includes the operation sequence
and the roles the quantities play (e.g. "part + part = whole, solve
for a part"); for a bare equation it is the equation's structure
with literals abstracted. Entity abstraction is what makes
"apples → oranges" a cosmetic change: the signature ignores the
entity, so the slot comparison is what catches it — the numbers
didn't change, so it fails on slots, not on structure.

**Strengths:** this is the only surveyed mechanism whose pass/fail
boundary lines up with the requirement's boundary. It *wants*
same-template-different-numbers and *rejects*
same-template-same-numbers, which is precisely §1a. It is
deterministic and explainable: a failure can name the reason
("same template, same numbers") instead of reporting a score.

**Weaknesses:** it needs a parser that understands the problem's
structure, and parser coverage is the failure mode. Bare equations
and standard arithmetic/equation word problems parse well; geometry
diagrams, multi-part problems, and free-form story problems parse
partially or not at all. A problem the parser cannot structure
must not silently pass — it falls through to the secondary
mechanisms with a visibly degraded verdict (§3, §4). Template
abstraction also needs care at the edges: over-abstracting
collapses genuinely different problems into one signature (false
rejects); under-abstracting treats cosmetic reorderings as new
templates (false accepts). The abstraction rules are part of the
fixture-validated design (§6), not an implementation detail to
improvise later.

### (iii) Symbolic normalization for math

**How:** parse mathematical expressions/equations in A and C into
an abstract syntax tree and reduce each to a canonical form before
comparing: sort commutative operands, normalize associativity,
reduce numeric constants and fractions, normalize variable names,
convert units to a common base, and expand-or-factor consistently
(one chosen direction, applied to both sides). Then:

- compare canonical forms **with literals abstracted** → the
  template signature of (ii), robust to 2 + 3x vs 3x + 2 and to
  2(x + 3) vs 2x + 6 (the same problem in two notations, correctly
  identified as the same template);
- compare the **literal multisets** under the canonical form → the
  instance check: same canonical template + same literals (up to
  the normalization's equivalences) = restatement.

**Strengths:** this is what makes (ii) actually work for symbolic
math instead of working only for word problems with obvious slots.
It turns the hardest cosmetic disguises — reordering, renaming,
unit conversion, equivalent notation — from false accepts into
detected restatements. It is fully local, deterministic, and
free.

**Weaknesses:** coverage, again — normalization is only as good as
the parser, and it covers the algebra/arithmetic curriculum, not
every problem shape Sam will meet (proofs, constructions, data
interpretation, problems whose meaning lives in a diagram). A
normalizer that claims coverage it doesn't have is worse than one
that reports its limits: unparseable input must degrade visibly
(§4), not pass quietly. There is also a genuine over-normalization
risk: canonical forms that erase pedagogically meaningful
differences (e.g. collapsing a factored and an expanded form when
the *skill being tested* is factoring) — the normalization rules
must be validated per topic area against fixtures, and a
normalization step that erases the tested skill is a bug, not a
simplification.

**Relationship to (ii):** (iii) is not a rival to (ii); it is (ii)'s
engine for symbolic content. The design treats them as one primary
mechanism: *normalize, then compare template + slots* (§3).

### (iv) Other mechanisms considered

**Token / n-gram overlap.** Cheap, exact, and excellent at one thing:
catching verbatim or near-verbatim copies, including a candidate
that lifts A's wording with new numbers pasted in (the word overlap
stays high even though the numbers changed — a useful complement
to the numeric check when the parser fails). It fails on paraphrase
by design, and it false-alarms on boilerplate ("Find the value of
x", "Show your work") unless stop-phrased. Verdict: include as a
cheap secondary signal, never as the gate — it catches the lazy
restatement and nothing else, and on its own it would bless every
paraphrased copy.

**Numeric-slot multiset comparison (parser-free).** Extract all
numbers from A and C (integers, decimals, fractions, percentages,
with unit normalization: 3 m vs 300 cm count once) and compare the
multisets, ignoring order. This catches "same numbers, cosmetic
changes" even when full structural parsing fails — it is the
fallback that keeps the *instance* half of the check alive for
unparseable problems. Its weakness is small-number coincidence:
classroom problems reuse 2, 3, 5, 10 constantly, so a multiset of
{2, 3} matching by chance is common. Mitigation, to be validated
on fixtures: weight by distinctiveness (rare/large/specific numbers
count more than small integers), and treat a multiset match as a
*fail* only combined with high topical/word overlap, or as a
*flag for regeneration* rather than a hard fail when the parser is
absent. Alone it is too blunt; paired with overlap it is a good
fallback.

**Answer-collision check (solve C, compare with A's answer).**
For arithmetic/algebra problems a solver can compute C's answer
and compare it to A's answer (which the tutor must obtain or
derive — see the open question in §7). A candidate whose answer
equals A's is suspicious: same-template problems with genuinely
different numbers rarely share an answer by chance, and an
answer-equal candidate leaks A's answer even if its numbers
differ. But answer equality is not proof of restatement (2 + 5 and
3 + 4 both equal 7 and are legitimately different practice), so
this is a *signal*, not a gate: answer-equal candidates should be
regenerated when cheap, and flagged when served. The deeper
question is policy, not mechanism: does internally solving A —
never surfacing the result — violate "never answers the assigned
problem"? The design's position is that an internal gate value
that is never presented is not an answer *given to Sam*, but that
is Noah's call, not the doc's (§7). If he says no, this signal is
dropped and the primary mechanism stands without it.

**LLM-as-judge.** Ask a model "is C a restatement of A?" This is
the most flexible mechanism — it handles paraphrase, word problems,
and edge cases the parser cannot — and the weakest fit for this
estate:

- **Cost:** the standing household preference is no per-call API
  spend for always-on personal infra (ululuth + Hetzner only;
  docs/RAG-EVALUATION.md criteria). A judge call per generated
  problem is exactly that spend, on the tutor's hottest path.
- **Placement:** generation is the personas' job, not the index's
  (RAG-EVALUATION.md, out of scope). A judge inside the check
  would smuggle a generation-shaped dependency into a gate that
  should be deterministic — and a gate whose verdicts are
  non-reproducible cannot be fixture-validated (§6) in any strong
  sense.
- **Local-model variant:** running a small local model (e.g. via
  Hermes tasks) avoids the API spend but trades it for resident
  compute on shared ARM hardware, model management, and verdicts
  that are still non-deterministic and uncalibrated. It is a
  separate question, explicitly out of scope for the index design,
  and this doc does not open it for the check.
- Verdict: **not recommended**, in either form, for the gate. The
  one legitimate use is *offline audit*: sampling served pairs
  after the fact for human review during validation (§6), where
  non-determinism costs nothing and flexibility helps find the
  parser's blind spots. The gate itself stays deterministic.

## 3. Recommendation

**Primary mechanism: normalize, then compare template + slots**
(§2ii + §2iii as one mechanism, in the future `tutor` package):

1. Parse A and C; normalize expressions/equations to canonical
   form (commutative sorting, constant/fraction reduction,
   variable-name and unit normalization, one consistent
   expand-or-factor direction).
2. Abstract numeric literals and entity names → template signature.
3. Compare: same signature + same literal multiset (under
   normalization) → **fail (restatement)**; same signature +
   different literals → **pass (desired similar)**; different
   signature → **weak pass** (different problem; acceptable only
   if it exercises the target skill, and recorded as such so
   generation prefers same-signature candidates).
4. For word problems the same logic applies with entity/role
   abstraction in place of symbolic normalization; where a problem
   is only partially parseable, the parsed part gates and the
   unparsed part is reported in the verdict.

**Secondary / fallback mechanism: numeric multiset + token overlap
+ embedding coarse gate, combined** — used when the primary
mechanism cannot parse a problem, and as a backstop audit on
primary passes:

- numeric multiset match (unit-normalized, distinctiveness-weighted)
  **and** high token overlap → fail (the paraphrase-proof
  restatement the parser missed);
- embedding distance in the "far" direction → off-topic flag (a
  generation failure, handled by regeneration, not a restatement
  failure);
- embedding distance in the "near" direction → *no verdict on its
  own* (see §2i: desired-similar and forbidden-restated live in
  the same band). It may corroborate a primary fail, never cause
  one alone.

**Explicit failure modes and how each surfaces to Sam:**

| Failure | Direction | What Sam sees (never silent) |
|---|---|---|
| Parser misses a disguised restatement (unparseable word problem, diagram-dependent problem) | **False accept — leaks the answer.** The worst failure. | Nothing at serve time can undo it — which is why the fallback runs on every primary pass and why verdicts are recorded (§4): the leak is found in audit, the parser gap is filed as a fixture, and the fallback is tightened. Mitigation at serve time: when the primary mechanism reports low parse confidence, Sam sees the degraded form (§4) instead of a clean walkthrough. |
| Over-normalization collapses distinct problems (factoring skill erased, distinct templates merged) | **False reject — blocks legitimate practice.** | Sam does not see a rejection. He sees a *different* similar problem (regeneration, §4) or, after the attempt budget, a worked example from his notes with its deep link. A blocked candidate is never explained as "rejected" — the tutor simply serves the next-best legitimate thing, and the rejection is recorded for fixture review. |
| Small-number coincidence (numeric multiset matches by chance) | **False reject.** | Same as above: regenerate; if the fallback keeps flagging, serve a worked example. The distinctiveness weighting (§2iv) exists to keep this rare. |
| Embedding gate misfires (near-band or far-band error) | Either, depending on direction. | The embedding signal never fails a candidate alone, so its misfires cannot block or leak by themselves; they can only trigger a regeneration or an audit flag. |
| Answer coincidence (different numbers, same answer) | **Soft false accept — partial leak.** | If the answer-collision signal is enabled (§7): regenerate once when cheap; if served anyway, no special surfacing (the answer is not named as A's answer), but the collision is recorded. If the signal is disabled by Noah: accepted as a known, recorded limitation. |

The surfacing rule, in one line: **Sam never sees a check verdict,
a score, or a rejection — he sees either a clean similar problem,
or a visibly degraded fallback (worked example with deep link, or
an honest "I don't have a clean similar problem for this one"
plus the pointer), and the verdict itself lives in the store
where the next run and the next review can see it.** That is the
development-orchestration skill's "degraded visibly" rule applied
to the tutor: the degradation is visible in *what is served* (a
worked example is visibly not a composed problem) and in the
record, not in a warning banner lecturing Sam about the gate.

## 4. Place in the tutor flow

**Where it runs: a generation-time gate, before the problem reaches
Sam, inside the `tutor` package (orchestration layer).** It is not
a post-hoc filter, not a student-facing tool, and not part of the
index MCP — the index's tools supply the inputs (worked examples
to generate from, passages to ground a paraphrase, resources for
the fallback pointer) and the check consumes them through the MCP
contract like any other tutor client. Concretely, the math path is:

1. Sam states a problem (verbatim paste or paraphrase) and asks
   for help.
2. Tutor classifies: math → similar-problem path; knowledge →
   pointer path (§1b).
3. Math path: tutor pulls worked-example material for the topic via
   `worked_examples(topic)` — honoring `match_quality`: a `weak`
   envelope means "I don't have material on that" and the path
   stops there rather than presenting the plausible-but-wrong top
   hit (the contract's rule, inherited, not re-decided here).
4. Tutor generates candidate C from that material.
5. **The check runs on (A, C).** Pass → walk through C fully, then
   hand A back to Sam. Fail → §4 failure chain below. The check
   never runs after the walkthrough has started, and a failed
   candidate's walkthrough is never shown "with a warning" — a
   warned restatement is still a restatement.
6. The verdict (pass/fail/weak-pass, mechanism used, parse
   confidence, the reference text's identity) is recorded in the
   store alongside the session state, per the data discipline.
   Recorded verdicts are what make §6's audit and the parser's
   improvement loop possible.

**What it consumes — and the paraphrase problem.** The check's
reference A is *the assigned problem text as Sam states it*. That
is often not the original: Sam paraphrases, drops numbers,
reorders, omits context ("the triangle one, find the missing
side"), or states only his attempted answer. Each mechanism
weakens differently under that input:

- **Numeric multiset:** weakest — dropped or altered numbers make
  the reference multiset wrong, so a restatement can pass (its
  numbers match the *paraphrase's* numbers, which are incomplete)
  and a legitimate problem can fail (its numbers match a number
  Sam misremembered).
- **Template/symbolic:** weakened proportionally to what was
  dropped — a partial template still gates on the parsed part, but
  an unparseable paraphrase ("the triangle one") yields no
  template at all.
- **Embedding:** shifted but not destroyed — a paraphrase embeds
  near the original's topic, so the coarse off-topic direction
  still works while the near direction stays unusable (as before).
- **Token overlap:** nearly useless on paraphrase, by design.

**Mitigations (designed in, not hoped for):**

- *Grounding attempt:* before checking, the tutor tries to ground
  Sam's statement against the corpus via `search_passages` — if a
  passage matches the stated problem strongly (the assignment
  sheet, the textbook problem, a worked example identical to A),
  the grounded text becomes the reference A alongside the
  paraphrase, and C is checked against **both**. A fail against
  either is a fail. Grounding is best-effort: when it finds nothing
  (common — assignments often aren't in the corpus), the check
  proceeds against the paraphrase alone and the verdict records
  "reference = ungrounded paraphrase, lower confidence".
- *Lower-confidence verdicts degrade visibly:* a pass on an
  ungrounded paraphrase with low parse confidence does not earn a
  clean composed walkthrough as its only option — the tutor may
  still serve it (a tutor that refused every paraphrased problem
  would be useless), but the fallback chain is shorter and the
  verdict is flagged for audit. The exact confidence-to-behavior
  mapping is a build-time calibration from §6's fixtures, not a
  number this doc invents.
- *Never ask Sam to paste the original as a precondition:* the
  tutor works with what Sam gives it. If Sam's statement is too
  thin to check against ("the triangle one"), the tutor's move is
  a clarifying prompt in the student persona's style (a prompt
  that makes Sam do the checking — "which problem, and what are
  its numbers?"), not a refusal and not an unchecked guess.

**What happens on failure (the chain, in order):**

1. **Regenerate with different parameters** — new numeric seed,
   new context/entities, and, when the failure names a cause
   (same numbers, same template), an instruction to vary that
   cause. Bounded: the attempt budget is an open question (§7);
   the design assumes a small bound (a handful, not an unbounded
   loop — a tutor that spins silently while Sam waits has failed
   differently).
2. **Fall back to a worked example from the index** —
   `worked_examples(topic)` with `match_quality` strong, cited by
   deep link, walked through as *someone else's* worked problem
   (which is what it is). This is the honest degraded form: Sam
   still gets a worked pattern to apply, and the tutor has not
   composed a restatement. A `weak` envelope here ends the chain
   with "I don't have material on that" — never a weak top hit
   dressed as a worked example.
3. **Escalate to Guardian — only on repeated failure, never on a
   single failure.** A single failed candidate is a normal
   generation miss; Sam gets the worked example and moves on. If
   the chain fails repeatedly on the same assignment/topic (the
   repetition threshold is Noah's, §7), the Guardian surface gets a
   declarative note ("Sam asked for help on [topic] and the tutor
   couldn't compose a clean similar problem — the worked example
   from [deep link] was served instead"), because a repeated
   failure is evidence about the material or the topic, not about
   one generation. Guardian escalation is a notification, not an
   intervention in Sam's session: the Guardian does not solve the
   problem for him by another route.

Study-mode and repetition items ride the same gate with the
reference adjusted (§1b): against the known assigned item when
one exists, otherwise against the just-solved item for
self-repetition, with the verdict recording which reference was
used.

## 5. Validation plan (before the tutor ships)

The mechanism is proven on fixtures **before** it gates a real
session, and the proof is recorded — not asserted in chat. The
plan:

**Fixture set.** Hand-built pairs (A, C) with a known-correct
verdict, in these categories (each category needs enough pairs
that a rate computed over it means something — the exact count is
set when the fixtures are built, alongside the acceptance
thresholds below):

- identical (verbatim copy) → must fail;
- cosmetic restatement: entity rename, sentence reorder,
  commutativity reorder, variable rename, unit conversion,
  equivalent notation (2(x + 3) vs 2x + 6) → must fail;
- paraphrase restatement (same numbers, new wording) → must fail;
- same template, different numbers (the desired outcome) → must pass;
- same template, different numbers, same answer (the edge case,
  §2iv) → verdict recorded per the answer-coincidence policy
  Noah sets (§7);
- different problem, same topic → must pass (and should be a
  weak pass if the signature differs);
- different topic → must pass the identity gate and fail the
  off-topic direction (the embedding coarse gate should catch it
  as generation drift);
- unparseable / diagram-dependent problems → must produce a
  degraded (fallback-path) verdict, never a silent clean pass;
- paraphrase-input variants of the above (A given as a student
  paraphrase with dropped numbers) → verdicts recorded with the
  weakened-confidence expectation, to measure how much the
  paraphrase input actually costs.

Fixture sources: real corpus topics first (Pythagorean, linear
equations, and whatever the corpus census shows Sam actually
studies — fixtures must agree with ground truth, per the coding
standard's reconciliation rule), synthetic pairs second for
categories the corpus doesn't cover. Fixtures live with the tutor
package's tests when it is built; the fixture *design* (categories
and verdict rules) is fixed here so the build cannot quietly
redefine "pass".

**Measurements.** Over the fixture set, report per category and
overall: false-accept rate (restatements passed — the leak rate),
false-reject rate (legitimate similars blocked), parser coverage
(fraction of fixtures the primary mechanism could structure), and
fallback-path rate (fraction forced to the secondary mechanisms).
Report the embedding signal's per-class cosine distributions
alongside, so the threshold derivation (§2i) is auditable. A
human audit sample — served (A, C) pairs reviewed after the fact,
optionally with an LLM-as-judge as an *offline* second opinion
(§2iv) — catches parser blind spots the fixtures didn't imagine.

**Acceptance thresholds — deliberately not set here.** This doc
does not invent numbers as settled: thresholds are set when the
fixtures exist and the measured distributions are visible, by
Noah, with the asymmetry stated now as the deciding principle:
**a false accept (leak) costs more than a false reject (block)**,
because a leak does the graded work while a block costs Sam one
regeneration or a worked example. The threshold-setting step must
therefore weight false accepts more heavily — how much more is
Noah's, and the measured rates are the input to that call, not
the other way around. Re-run the fixture suite (and re-derive any
embedding constants) when the corpus grows materially, the
embedding model changes, or a new topic area's parser is added —
the same re-derivation rule the contract states for
`match_quality`.

**Build-time gates this design hands forward (not decisions made
here):** when the tutor package is built, the coding standard's
gates apply to it (≥90% unit coverage, mypy strict, 500-LOC cap,
named patterns, DI/strategy registries for the check mechanisms —
the primary and fallback mechanisms are strategies behind one
check interface, selected by registry, not by if/else chains).
The dispatcher re-runs those gates independently before accepting
the milestone; this doc's fixture suite is an input to that
verification, not a substitute for it.

## 6. What this design deliberately does not do

- It does not change the index MCP, its tools, or the contract.
  `match_quality` is a *topic-relevance* signal; similarity is a
  *candidate-vs-assigned* signal. Conflating them — e.g. reading a
  "strong" pointer as evidence a candidate is safely dissimilar —
  is an error this design names so the build doesn't make it.
- It does not settle the RAG stack decision (Noah's, in
  docs/RAG-EVALUATION.md). The embedding secondary uses whatever
  embedding provider the estate settles on; its constants are
  provider-specific and re-derived on change. The primary
  mechanism does not depend on the RAG decision at all — that
  independence is a feature, not an accident.
- It does not cover D-1 (deferred) or the profile-birth staging,
  and it touches no live container, gateway route, or token.

## 7. Open questions for Noah (his, not decided here)

1. **Internal solving of A for the answer-collision signal:** may
   the tutor compute A's answer internally, never surfacing it,
   solely to compare against C's answer — or does "never answers
   the assigned problem" forbid even internal solving? If
   forbidden, the answer-collision signal is dropped (§2iv).
2. **Answer coincidence policy:** a candidate with different
   numbers but the same answer as A — fail, regenerate-once, or
   serve-and-record? (Recommendation in this doc: regenerate when
   cheap, serve-and-record otherwise — but the policy is his.)
3. **Attempt budget and Guardian threshold:** how many
   regenerations before the worked-example fallback, and how many
   repeated chain failures on one assignment/topic before the
   Guardian note fires? (§4 assumes small numbers; the numbers
   are his.)
4. **False-accept vs false-reject tradeoff:** once §6's fixture
   rates are measured, what false-accept rate is acceptable at
   ship, and how much more heavily than false rejects should it
   be weighted? (The asymmetry principle is stated in §5; the
   numbers are his.)
5. **Confidence-to-behavior mapping for paraphrase input:** when
   the reference is an ungrounded paraphrase with low parse
   confidence, should the tutor still serve a composed
   walkthrough, or always degrade to the worked example? (§4
   leans serve-with-shorter-fallback; the call is his.)
6. **LLM-as-judge, permanently out or audit-only forever?** This
   doc recommends audit-only (§2iv). Confirm that no future
   tutor step should promote a judge to the gate — or name the
   condition (e.g. a local model landing on hal9k for another
   reason) under which that would be revisited.
