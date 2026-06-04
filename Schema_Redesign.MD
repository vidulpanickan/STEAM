# Clinical Assertion Schema — Design Rationale

A compact, linguistically grounded annotation schema for extracting **context/assertion**
from English EHR free-text, designed to be emitted by a small fine-tuned LLM in
resource-constrained / edge deployments.

## Objective: linguistic interpretation, not clinical interpretation

We capture **only what the text linguistically asserts** — readable from cue words, scope,
syntax, and the semantics of framing verbs — and we capture it *source-relative*: the
author's expressed commitment at the moment the note was written.

We deliberately do **not** annotate clinical interpretation (active vs. resolved, normal
vs. abnormal, whether a denial is truthful). Clinical interpretation is ambiguous in many
cases, depends on external medical knowledge, and evolves with diagnostic criteria — so it
is left to the clinician and to a separate **deterministic projection layer** downstream.
Our job is to hand that layer (and the doctor) a complete set of orthogonal linguistic axes
from which any clinical reading can be derived.

Worked example — *"BP is 180/110"*: the text asserts a **measurement event, present tense,
full certainty**. It does **not** assert "hypertension" or "abnormal." We annotate the
former; the doctor/projection derives the latter.

## Design principles

- **Linguistic basis, clinically complete.** Each axis is grounded in a linguistic
  construct, and together the axes cover everything a clinician needs to reconstruct
  clinical status.
- **One axis = one clear question, finite answers.** Task clarity for the LLM is maximized;
  every axis answers a single well-defined question with a small closed value set.
- **Axes are orthogonal.** The value on one axis places zero constraint on any other; every
  combination is a coherent linguistic state.
- **Values are orthogonal too.** Vocabularies do not collide or overlap across axes — an
  emitted token unambiguously identifies both its axis and its value.
- **Minimal, small-token values.** Verbalizers are chosen as single-token, high-frequency,
  semantically transparent words — selected for what the model reliably maps the class onto,
  not for fidelity to any annotation-corpus tradition.

## The axes (5)

| Axis | Key | Values | Default | Question it answers |
|---|---|---|---|---|
| Negation | `negated` | `yes` / `no` | `no` | Does the author deny the entity? |
| Certainty | `certainty` | `certain` / `likely` / `maybe` | `certain` | How strong is the author's epistemic commitment? |
| Realis | `realis` | `actual` / `hypothetical` | `actual` | Is occurrence asserted, or only entertained/contingent? |
| Experiencer | `experiencer` | `self` / `family` / `other` | `self` | Who is presented as undergoing it? |
| Temporality | `temporality` | `past` / `now` / `future` | `now` | When, by grammatical tense relative to note time? |

### Per-entity output

```json
{
  "entity": "pulmonary embolism",
  "negated": "no",
  "certainty": "maybe",
  "realis": "actual",
  "experiencer": "self",
  "temporality": "now"
}
```

### Hard conventions

- Negation is strictly binary; **every hedge routes to Certainty**, never to Negation.
- `negated: no` = affirmed/present; `negated: yes` = denied/absent. (Default flips to `no`
  because most findings are not negated — guard against silent polarity inversion.)
- "ruled out" → `negated: yes` + `certain`; "cannot rule out" / "?PE" → `negated: no` + `maybe`.
- Conditionals: irrealis "if" → `realis: hypothetical`; real-but-triggered ("dyspnea on
  exertion") → `realis: actual` (the trigger is **not** encoded — it survives as captured
  text for projection).
- Experiencer: primary patient → `self`; named genetic relative → `family`; everyone else
  (spouse, roommate, donor, fetus, generic) → `other`.
- Temporality is **grammatical tense, not clinical chronicity** ("history of MI" → `past`
  regardless of whether it resolved).
- No axis other than Negation uses `yes`/`no`.

## How this lines up with linguistic studies

The five axes are not ad hoc; each maps to an established linguistic construct, and the
*decomposition* itself follows the formal-semantics treatment of event factuality.

- **Negation** is sentential **polarity** in formal logic, and corresponds to the negation
  detection task pioneered for clinical text by NegEx (Chapman et al.). We key it on the
  *negative pole* so the label never self-contradicts.
- **Certainty** is **epistemic modality** (Lyons; Palmer) — the speaker's evaluation of the
  truth of a proposition — operationalized over clinical hedging ("likely," "consistent
  with," "cannot exclude").
- **Splitting factuality into Negation × Certainty × Realis** follows the **FactBank /
  event-factuality** tradition (Saurí & Pustejovsky), where factuality is modeled as the
  *combination* of polarity and modal strength rather than a single flat category. This is
  also why we reject a single bundled "modality" axis: it would re-entangle dimensions that
  the literature treats as independent, breaking orthogonality.
- **Realis** is the cross-linguistic **realis/irrealis mood** distinction (Mithun; Palmer,
  *Mood and Modality*): whether an event is asserted as actualized or merely entertained
  (conditionals, recommendations, counterfactuals). We name the axis `realis` rather than
  "mood" specifically to avoid the clinical homonym (affective *mood*).
- **Experiencer** is the **thematic/semantic role** of that name (Fillmore-style case
  grammar) and is taken directly from the ConText algorithm's experiencer feature (Harkema
  et al.) — chosen over "subject" to avoid grammatical-subject ambiguity.
- **Temporality** is **deictic tense** anchored to the **deictic center** / reference time
  (Reichenbach), again following ConText — deliberately *tense*, kept distinct from clinical
  chronicity.

## Key design decisions (and one we deferred)

- **Verbalizers are model targets, not annotation labels.** Canonical axis names are for
  annotators/validators; the model emits single tokens under a frozen bijection, so the two
  can be optimized — and ablated — independently.
- **`self` not `patient`** for the primary experiencer: avoids multi-token splits and the
  "be patient" polysemy; `self`/`other` is a clean pretrained dichotomy.
- **`negated` not `affirmed`/`asserted`** as the key: keying on the positive pole makes the
  negative value self-contradictory (`asserted: no` ≈ "not asserted" vs. "asserts absence").
  Keying on the negative pole keeps values clean.
- **`realis` not `mood`**: avoids the clinical affective-mood collision.
- **Dropped — Source/Evidentiality** (Aikhenvald): patient-reported vs. clinician-observed.
  This is the one omission with real downstream cost; **re-add it** (`seen`/`told`) if any
  cohort definition or decision-support rule weights provenance.
- **Dropped — Conditionality and Genericity**: the trigger and class-level distinctions
  collapse into captured text / the `hypothetical` bucket. Re-add Conditionality
  (`free`/`bound`, realis-only) only if a downstream rule branches on the trigger itself.

## Deferred to the projection layer

Active/resolved status, normal/abnormal, true clinical chronicity, trigger recovery
("on exertion"), and (currently) provenance weighting — all derived deterministically from
the orthogonal axes above, not annotated.

## Open / settle empirically

- **`hypothetical` tokenization** — verify single-token in the target tokenizer (bare and
  leading-space); fall back to `if` if it fragments. The only value with a possible hard
  failure.
- **Certainty cardinality** — keep 3-way, but collapse to 2-way `certain`/`maybe` if
  inter-annotator agreement on the `likely`/`maybe` boundary fails in a pilot.
- **Tokenizer whitespace** and a **model-prior probe** (does the model already map each class
  onto the chosen token?) before freezing.
