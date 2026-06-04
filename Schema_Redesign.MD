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
- **Axes are orthogonal.** The value on one axis places (almost) zero constraint on any
  other; nearly every combination is a coherent linguistic state. (One residual coupling is
  documented below: under `realis: hypothetical`, `certainty` is read off the asserting
  clause and is otherwise vacuous.)
- **Values are orthogonal too.** Vocabularies do not collide or overlap across axes — an
  emitted token unambiguously identifies both its axis and its value. (Whether this property
  is *necessary* depends on the emission format — see "Emission format" under Open / settle.)
- **Minimal, small-token values.** Verbalizers are chosen as single-token, high-frequency,
  semantically transparent words — selected for what the model reliably maps the class onto,
  not for fidelity to any annotation-corpus tradition.

## The axes (5)

| Axis | Key | Values | Default | Question it answers |
|---|---|---|---|---|
| Negation | `negated` | `yes` / `no` | `no` | Does the author deny the entity? |
| Certainty | `certainty` | `certain` / `likely` / `maybe` | `certain` | How strong is the author's commitment to the *polarized* claim? |
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

- **Certainty grades the proposition *as polarized by* `negated` — not the bare finding.**
  So `negated: yes` + `likely` means "**probably absent**," not "probably present." Stating
  this is mandatory: without it, two annotators silently split on whether certainty attaches
  to the finding or to its absence, and agreement on the axis collapses for a reason that
  looks like annotator noise but is actually a hole in the guideline. The convention is the
  FactBank polarity × modality cross-product:

  | `negated` | `+ certain` | `+ likely` / `maybe` |
  |---|---|---|
  | `no` (present) | definitely present (CT+) | probably / possibly present (PR+ / PS+) |
  | `yes` (absent) | definitely absent (CT−) | probably / possibly absent (PR− / PS−) |

- **Realis is triggered by *irrealis only* — not by futurity, and not by "being a
  recommendation."** The test for `hypothetical` is an **unactualized condition** (an "if,"
  a counterfactual, a contingent/conditioned suggestion). A flat, *issued* plan or order is
  `realis: actual`; its not-yet-ness lives in `temporality: future`. Futurity alone is never
  sufficient for `hypothetical`.
  - "if she develops chest pain, return" → `hypothetical` + `self` + `future` (chest pain is
    the trigger condition, **not** an asserted finding)
  - "take aspirin daily" / "will start metoprolol" → `actual` + `future` (a real, issued plan)
  - "if symptoms worsen, start prednisone" / "would recommend aspirin if tolerated" →
    `hypothetical` (the action is conditioned)
  - real-but-triggered ("dyspnea on exertion") → `actual` (the trigger is **not** encoded —
    it survives as captured text for projection)

- **Experiencer:** primary patient → `self`; named genetic relative → `family`;
  everyone/**everything** else → `other`. `other` is the deliberate catch-all for any
  non-`self`, non-genetic-family reference, **explicitly including non-persons, generic /
  teaching statements, and template / boilerplate cues** (spouse, roommate, donor, fetus,
  "patients with CHF…", eligibility/criteria lists). Routing these to `other` keeps them off
  the patient's problem list. Two limits, both accepted by design:
  - `other` does **not** separate a real third party from generic boilerplate (this is the
    dropped genericity distinction).
  - `other` does **not** reach patient-*directed* content such as instructions ("take aspirin
    daily" is `self`); that case is handled by `realis`/`temporality`, not experiencer. See
    Worked mappings.

- **Temporality is grammatical tense, not clinical chronicity** ("history of MI" → `past`
  regardless of whether it resolved). Conventions for the common tenseless cases in EHR text:
  - tenseless telegraphic fragments ("pt c/o CP") default to `now`
  - bare imperatives / directives ("take aspirin," "return if…") default to `future` — the
    directive mood, not surface tense, carries the time
  (In practice much of EHR text is telegraphic, so `temporality` is **default-carried** far
  more often than the "grammatical tense" framing alone implies.)

- No axis other than Negation uses `yes`/`no`.

### Worked mappings for tricky inputs

Input types that look like they need a new axis or value but don't — they fall onto the
existing five. Filtering of *non-assertions* is done by structure/segmentation upstream, not
by the axes.

- **Standalone lab value** — "BP 180/110," "K 5.2": a measurement event →
  `no` / `certain` / `actual` / `self` / `now`. The number is on no axis; it survives in
  captured text and the projection derives "high/abnormal." (Extraction caveat: a bare table
  row may not parse into a clean entity — an OCR/segmentation problem, not a schema gap.)
- **Patient instruction** — "return if you develop chest pain": `hypothetical` (the "if") +
  `self` + `future`. Chest pain correctly does **not** surface as a present finding.
- **Issued order** — "take aspirin daily": `actual` + `future`. A real plan, **not**
  hypothetical.
- **Generic education** — "patients with CHF should limit salt": `experiencer: other`
  (generic). Off the problem list because it is about patients-in-general, not this patient.
- **Template / boilerplate family cue** — pre-printed checklists or criteria lists with a
  family-sounding cue: `experiencer: other`. (This is the MITRE immunization-criteria
  false-positive: the axes route attribution safely but cannot tell it from a real relative —
  the genericity gap.)
- **Unanswered / blank item** — "Smoking: ☐," "PHQ-9: [blank]": asserts nothing. Because
  there is **no abstention value**, a blank pushed through the axes default-fires into a
  fabricated positive (blank smoking box → "current smoker, certain, actual, self, now").
  Blanks and other non-assertions must be **filtered upstream (segmentation)** before the
  parser, or gated by an optional `assertion_present: no` escape (see Known gaps). They must
  not reach the axes.

### Input contract

- The model labels **one marked target entity at a time**, given a context window around it.
  The LLM-era evidence puts the sweet spot at roughly **±2 sentences**.
- Coordinated lists produce **one record per entity**: "no fever, chills, or cough" → three
  entities, each `negated: yes`. Distributing polarity/certainty correctly across coordinated
  heads is a **scope** problem — the dominant documented failure in this task (NegEx ≈ 58% on
  bare "not"; dependency-based successors only partially recover it). The axes can *represent*
  every correct answer; correct *assignment* is bought with **training data**, not schema
  design. Build coordinated-list / scope-terminator / nested-scope cases and over-weight them
  relative to natural frequency.

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
  also why we reject a single bundled "modality" axis and why we do **not** merge Negation
  and Certainty: a merged axis cannot represent a *hedged negative* ("probably no PE"), which
  is common in notes, and it re-entangles dimensions the literature treats as independent.
  (Note: ConText itself had no certainty axis; the authority for the polarity/modality split
  is FactBank. The one time the ConText lab tested a certainty feature — Chu et al. 2006 — it
  *lowered* accuracy, which is why we keep Certainty minimal and empirically validated.)
- **Realis** is the cross-linguistic **realis/irrealis mood** distinction (Mithun; Palmer,
  *Mood and Modality*): whether an event is asserted as actualized or merely entertained
  (conditionals, *conditional/counterfactual* recommendations, counterfactuals). Note: an
  **issued** plan is realis-`actual` + `future`; only an unactualized condition is irrealis —
  futurity itself is not. We name the axis `realis` rather than "mood" specifically to avoid
  the clinical homonym (affective *mood*).
- **Experiencer** is the **thematic/semantic role** of that name (Fillmore-style case
  grammar) and is taken directly from the ConText algorithm's experiencer feature (Harkema
  et al.) — chosen over "subject" to avoid grammatical-subject ambiguity.
- **Temporality** is **deictic tense** anchored to the **deictic center** / reference time
  (Reichenbach), again following ConText — deliberately *tense*, kept distinct from clinical
  chronicity.

## Key design decisions (and the ones we deferred)

- **Verbalizers are model targets, not annotation labels.** Canonical axis names are for
  annotators/validators; the model emits single tokens under a frozen bijection, so the two
  can be optimized — and ablated — independently.
- **`self` not `patient`** for the primary experiencer: avoids multi-token splits and the
  "be patient" polysemy; `self`/`other` is a clean pretrained dichotomy.
- **`negated` not `affirmed`/`asserted`** as the key: keying on the positive pole makes the
  negative value self-contradictory (`asserted: no` ≈ "not asserted" vs. "asserts absence").
  Keying on the negative pole keeps values clean. (`affirmed` would make the *key* sound
  friendlier at the cost of making the *values* ambiguous — a bad trade, since the model
  emits values.)
- **`realis` not `mood`**: avoids the clinical affective-mood collision.
- **Dropped — Source/Evidentiality** (Aikhenvald): patient-reported vs. clinician-observed.
  This is the one omission with real downstream cost, and it is the **top candidate for
  re-addition** (`seen`/`told`). A large share of clinical negation is patient-reported
  ("denies chest pain"), and the safety-gate use case protects a provenance-bearing span — so
  provenance is plausibly **more load-bearing than Certainty cardinality**. Re-add if any
  cohort definition or decision-support rule weights provenance.
- **Dropped — Conditionality and Genericity**: the trigger and class-level distinctions
  collapse into captured text / the `hypothetical` / `other` buckets. Re-add Conditionality
  (`free`/`bound`, realis-only) — or a `general` experiencer value — only if a downstream rule
  branches on the trigger/genericity itself; otherwise prefer upstream segmentation.

## Known gaps the schema does not close on its own

These are real and mostly fixed by **data, evaluation, or upstream segmentation** — not by
adding axes.

- **No abstention.** Every axis forces a value, so genuinely unlabelable cases (and blank
  form items) become noise or fabricated positives. Filter non-assertions upstream; optionally
  add an `assertion_present` flag. Note some boundaries are not reliably annotatable even by
  experts (e.g., pneumonia-from-report κ ≈ 0.085) — forcing a hard label there biases
  evaluation against uncertainty.
- **Genericity.** `other` + `hypothetical` keep boilerplate off the problem list but cannot
  distinguish a real third party / real contingency from generic template text. Prefer
  upstream segmentation of instruction/education/criteria text; or re-add Conditionality / a
  `general` value if a rule depends on it.
- **Scope & rare-value starvation are data problems, not schema problems.** Decomposition
  makes the *common* classes cleaner and each decision auditable, but it **relocates** the
  hard conditional/hypothetical (and `actual`-`future` vs `hypothetical`) boundary into
  `realis` rather than removing it — the literature's largest model gaps sit exactly there.
  Over-sample the rare values (`maybe`, `hypothetical`, `family`/`other`) and the scope cases;
  natural frequency starves them.
- **Cross-source transfer is not free.** A scheme validated on one note type need not survive
  another ("Negation's not solved," Wu 2014). Evaluate across sources, not just on one corpus.

## Deferred to the projection layer

Active/resolved status, normal/abnormal, true clinical chronicity, trigger recovery
("on exertion"), and (currently) provenance weighting — all derived deterministically from
the orthogonal axes above, not annotated.

## Open / settle empirically

- **`hypothetical` tokenization** — verify single-token in the target tokenizer (bare and
  leading-space); fall back to `if` if it fragments. The only value with a possible hard
  failure.
- **Certainty cardinality** — keep 3-way, but collapse to 2-way `certain`/`maybe` if
  inter-annotator agreement on the `likely`/`maybe` boundary fails in a pilot. **Validate
  model calibration too, not just annotator agreement**: models skew toward extreme responses
  and can collapse `likely` → `certain` even when annotators agree, so check certainty
  calibration explicitly on held-out data.
- **Realis boundary** — the `actual`-`future` (issued plan) vs `hypothetical` (conditioned)
  call is the hardest, data-starved distinction; build and over-sample minimal pairs
  ("take aspirin daily" vs "if X, take aspirin"; "dyspnea on exertion" vs "if dyspnea, …").
- **Emission format** — if emitting **keyed JSON**, the key identifies the axis and
  value-orthogonality is a safety margin, not a necessity. If a **compressed/keyless** format
  is used for edge token budget, value-orthogonality becomes load-bearing and near-collisions
  (`now`/`no`, `actual`/`hypothetical`) are live corruption risks with no key to catch them.
  Decide this before freezing the verbalizers.
- **Tokenizer whitespace** and a **model-prior probe** (does the model already map each class
  onto the chosen token?) before freezing.
