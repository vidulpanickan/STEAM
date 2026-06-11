# SCHEMA_PRO — Flat Per-Entity Clinical Schema (v3, selectable-axis LoRA target)

The single source of truth for the STEAM_PRO schema. It is a **fixed menu of flat, independent
fields** — canonical names, canonical order, mostly closed vocabularies. At inference, each request
**names the subset of fields it wants**; the model returns, per marked entity, a flat JSON object
containing **only the requested fields, in canonical order**. You pay generation tokens only for
what you ask for, and "which fields apply to this entity" becomes the **caller's** choice, not
something the model has to infer.

Why this is reliable (the earlier worry was varying shape): the output keys are **dictated by the
request** — fully deterministic, the model just obeys — not *inferred from entity type*, which was
the error-prone case. The guarantee holds **only if training includes varying requested subsets**
(§6); that is a training-format commitment, not just an inference trick.

Slot values are the **verbatim text span** copied from the note; unit/date normalization is a
separate deterministic pass, never the model's job. Every decision is referenced in §9; dropped /
out-of-scope items and citations in §10–11.

---

## 1. Record shape (container; stores the FULL label set)

The stored dataset keeps **every field labeled** per entity — the full menu. The builder samples
field subsets from it to create training examples (§6). Container is unchanged:

```json
{
  "text_id": "text_00000",
  "origin": "real",
  "source": "labelling",
  "group": "real",
  "text": "... Last dose of Antibiotics:  [E1] Cefipime [/E1] - [**2177-3-15**] 02:48 AM ...",
  "entities": {
    "E1": { /* full flat label — every menu field — see §2 */ }
  }
}
```

## 2. The field menu (canonical order, names, vocabulary)

`id` is the anchor — **always emitted** so the object maps back to the marker. The 12 fields below
are each independently requestable. A fully-labeled entity (and the maximal request) looks like:

```json
{
  "id": "E1",
  "type": "medication",
  "negated": "no",
  "certainty": "certain",
  "realis": "actual",
  "experiencer": "self",
  "temporality": "past",
  "status": "stopped",
  "severity": "unspecified",
  "value": null,
  "unit": null,
  "date": "[**2177-3-15**] 02:48 AM",
  "body_site": null
}
```

(`entity`, the verbatim marked span, is recoverable from the `[E1] … [/E1]` markers, so it is
optional in the output and normally omitted to save tokens.)

## 3. Inference — request a subset, get only those keys

The request lists the wanted `fields`; the model emits each entity with `id` + those fields, in
canonical order. `fields` omitted (or `"all"`) ⇒ emit the full menu.

```json
// user: { "text_id": "...", "text": "...[E1]...[/E1]...", "fields": ["negated","certainty"] }
// assistant:
[ { "id": "E1", "negated": "no", "certainty": "certain" } ]
```

```json
// user fields: ["value","unit","status"]
// assistant:
[ { "id": "E1", "value": "0.03", "unit": "mcg/Kg/min", "status": "none" } ]
```

One field-set applies to all entities in the note (global per request) — the common case. Per-entity
field requests are possible but not the default.

## 4. Fields — values, defaults, decision rule

The closed-vocab axes keep the RULES.md contract: **start from default, flip ONLY on an explicit
surface cue that syntactically scopes this entity; no clinical inference.** The span fields are
**`null` unless the value is explicitly written**, and hold the **verbatim span**.

**Closed-vocabulary axes:**

| field | values | default | cue |
|---|---|---|---|
| `type` | medication / lab / problem / procedure / other | *(from upstream)* | entity kind |
| `negated` | yes / no | **no** | negation operator (`no`, `denies`, `ruled out`, `without`) |
| `certainty` | certain / possible | **certain** | hedge word (`likely`, `possible`, `?`, `concerning for`) |
| `realis` | actual / hypothetical | **actual** | irrealis marker (`if`, `would`, contingent) |
| `experiencer` | self / family / other | **self** | attribution (blood relative→family; non-blood/template→other) |
| `temporality` | past / now / future | **now** | tense/time marker (`history of`→past; `will`/`plan`→future) |
| `status` | started / stopped / increased / decreased / none | **none** | med change verb (`started on`, `d/c'd`, `increased to`) |
| `severity` | mild / moderate / severe / unspecified | **unspecified** | severity word (`mild`, `severe`, `massive`, `trace`) |

**Span fields (verbatim span or `null`):**

| field | holds | example span |
|---|---|---|
| `value` | the entity's quantity — med dose amount, lab/vital result, problem stage | `40`, `14`, `109/50`, `IIIb` |
| `unit` | unit for `value` | `mg`, `mcg/Kg/min`, `K/uL`, `mmHg` |
| `date` | date/time tied to the entity | `[**2177-3-15**]`, `POD 2` |
| `body_site` | anatomical location | `RLL`, `left heel` |

Notes:
- `status` is the medication **change action** (CMED). Default `none` covers an explicitly
  *continued* med and no mention alike. `discontinued` → `status=stopped`, `temporality=past`,
  `negated=no`; `declined/refused/not done` → `negated=yes`, `status=none`.
- **No `course` axis.** Trend/resolution rides on `temporality`+`negated`.
- **Compound values stay as one raw span** (`value="109/50"`) — splitting is a downstream parse.
- **`value`+`unit` is generalized:** dose for meds, result for labs/vitals, stage for problems —
  one pattern, not per-type slots.
- **No `route` / `frequency`.** Dropped — the dose (`value`+`unit`) carries the medication quantity;
  route/sig are low-value med-reconciliation detail, easy to re-add to the menu if ever needed.

## 5. Which fields you'd typically request per `type` (caller / labeler guidance)

`type` no longer gates anything — it is just a field. This table is guidance for *which subset to
request* (and which to label), not a shape rule:

| type | usually request | rarely meaningful |
|---|---|---|
| `medication` | axes, `status`, `value`+`unit` (dose), `date` | `severity`, `body_site` |
| `lab` (incl. vital) | axes, `value`+`unit` (result), `date` | `status`, `severity` |
| `problem` | axes, `severity`, `value`+`unit` (stage), `body_site`, `date` | `status` |
| `procedure` | axes, `body_site`, `date` | `status`, `severity`, `value`, `unit` |
| `other` | axes only | the rest |

## 6. Generation & training format

Per `build_chat_dataset.py`, one or more examples per note:
- **system** = `FINAL_TRAINING_DATA/system_prompt.txt` (the schema + the "emit only requested
  fields, in canonical order" contract).
- **user** = `{"text_id": ..., "text": <note with [Ei] … [/Ei] markers>, "fields": [...]}`.
- **assistant** = a JSON **array** of `{id + requested fields}` objects, one per entity, in marker
  order.

**Selectable-axis training (required):** because the stored data carries the full label, the builder
emits **several examples per note with different `fields` subsets** — e.g. the full menu, the 5
axes, single axes, and type-appropriate bundles (`[value,unit,status]`, `[severity,body_site]`). The
model thus learns to honor the request exactly. Always-full training would make subset requests
off-distribution. Keep canonical order in every subset so the frame stays predictable.

## 7. Backward compatibility with the current data

- Today's `training_data_all.jsonl` labels 5 axes per entity. The full menu adds 7 fields: `type`,
  `status`, `severity`, `value`, `unit`, `date`, `body_site` (route/frequency are *not* added).
- Migration: keep the 5 axis values; add `type` (upstream; default `other`), `status=none`,
  `severity=unspecified`, span fields `=null`. No information loss — the new fields then get
  **labeled** in a separate pass.
- **Primary path is generative + LoRA**, this schema is its target. An encoder remains the natural
  fit for the closed-vocab axes (one head per field = selectable for free, zero generation); the
  span fields need the generative model or a span-tagger. A hybrid is viable.

## 8. Invariants (carried from RULES.md, enforced for every field)

1. Default + explicit cue for every axis; a default is never inferred.
2. `null` unless the value is written in the text for every span field; nothing imputed from world
   knowledge.
3. Span-faithful: span fields hold the **verbatim** text; normalization is a separate downstream pass.
4. Syntactic scope governs both axis flips and span attachment.
5. No interpretive fields computed from reference knowledge (the reason `abnormal_flag` was dropped).
6. The model emits **exactly the requested fields, in canonical order** — never more, never fewer,
   never reordered.

## 9. Decision log (each field/value → its source)

| field | values | default | source standard | note |
|---|---|---|---|---|
| `negated` | yes / no | no | i2b2-2010 assertion; NegEx | — |
| `certainty` | certain / possible | certain | i2b2-2010 (`possible`); cTAKES uncertainty | — |
| `realis` | actual / hypothetical | actual | i2b2-2010 conditional/hypothetical; i2b2-2012 modality | — |
| `experiencer` | self / family / other | self | i2b2-2010 not-patient; cTAKES subject | — |
| `temporality` | past / now / future | now | i2b2-2012 temporal; CMED temporality | absorbs resolved→past, resolving→now |
| `status` | started / stopped / increased / decreased / none | none | CMED Action set | dropped `continued`+`unchanged` → single `none` |
| `severity` | mild / moderate / severe / unspecified | unspecified | cTAKES severity; FHIR Condition.severity | — |
| `value`+`unit` | verbatim span | null | i2b2-2009 dosage; FHIR Observation.value/unit | generalizes med dose + lab result + stage |
| `date` | verbatim span | null | i2b2-2012 TIMEX3 | normalization downstream (MIMIC dates shifted) |
| `body_site` | verbatim span | null | cTAKES body location; FHIR bodySite | — |

**Design decisions (v3):**
- **Selectable-axis** — schema is a field menu; the request dictates the emitted subset (deterministic
  shape), trained with varying subsets. Minimal tokens; "conditional field" question moves to the caller.
- **Flat + span-only** — no `axes`/`slots` nesting, no `{span, normalized}` objects.
- **Generalized** — med `dose` and lab `value` merged into `value`+`unit` (+ problem stage).
- **Dropped** (see §10) — `route`, `frequency`, `indication` relation, `components[]` list,
  `reference_range`, `specimen`, `form`, `abnormal_flag`, `duration`, `scale_value`, `course` axis.

## 10. Out of scope / dropped (and why)

**Dropped to keep the menu lean and the target flat** (re-addable if an eval justifies it):
- `route`, `frequency` — low-value medication-reconciliation detail; the dose (`value`+`unit`)
  carries the medication quantity.
- `indication` — a cross-entity relation; references are the most error-prone generative output.
- `components[]` — a variable-length list (BP/ABG parts); kept as one raw `value` span instead.
- `reference_range`, `specimen`, `form`, `duration`, `scale_value` — long-tail; `scale_value` folds
  into `value`+`unit`.
- `abnormal_flag` — interpretive (high/low from a range) → violates invariant §8.5.
- `{span, normalized}` nesting — normalization moved downstream.

**Out of scope entirely** (specialty-registry universe): oncology biomarkers (PD-L1 CPS/TPS),
genomic mutation status, tumor response/progression dates, performance status as its own pipeline,
SDoH, device attributes, demographics. (Ordinal scales clinicians do chart — ECOG/NYHA/GCS/
Child-Pugh — degrade into `value`+`unit`, e.g. `value="II"`, `unit="NYHA"`.)

## 11. Evidence base & sources

- **Medication attributes**: i2b2-2009, MedEx, n2c2-2018, FHIR Dosage. v3 keeps the highest-value
  subset (`value`/`unit` = dose, `status` = change) and drops `route`/`frequency`/`form`/`duration`.
- **Medication change events** (`status`): CMED 2021/2022 Action {Start, Stop, Increase, Decrease,
  UniqueDose, Unknown}.
- **Assertion cluster** (the 5 axes): i2b2-2010 assertion, i2b2-2012 polarity+modality, cTAKES,
  medspaCy ConText, OMOP `term_modifiers`. Negation + certainty in every system.
- **`severity` / `body_site`**: cTAKES nine-attribute set + FHIR Condition.severity/bodySite.
  (`course`, cTAKES-only, dropped.)
- **Lab/Observation `value`/`unit`**: FHIR Observation.value[x]; NER shared tasks treat labs only as
  bare "test" concepts.
- **Clinician information-needs**: Assessment & Plan dominates chart review (Brown 2014 67%; Clarke
  2014 92% of PCPs); discharge studies (Chatterton 2023; Sorita 2021; JGIM 2025) rank med changes +
  dates highly — motivating `status`, `value`, `date`.
- **Recent LLM-IE (2023-26)**: Agrawal EMNLP-2022 (few-shot med-attribute + med-status schema, and
  instruction-conditioned extraction — the basis for selectable-axis prompting); Microsoft UMA 2025;
  Flatiron. Unanimous caveat: the span alone is the *label*; normalization is a separate deterministic
  layer. Structured-output reliability scales with a fixed, flat schema + request-dictated keys.

*Key sources: i2b2-2009 PMC2995676 · MedEx PMC2995636 · n2c2-2018 PMC7489085 · CMED PMC10529825 ·
i2b2-2010 PMC3168320 · i2b2-2012 PMC3756273 · cTAKES Default Clinical Pipeline · FHIR
Observation/Condition/Dosage (build.fhir.org) · OMOP NOTE_NLP · Brown 2014 PMC4081746 · Clarke 2014
PMC3974234 · Chatterton 2023 PMC11169121 · Sorita 2021 PMID 28885382 · Agrawal 2022
aclanthology 2022.emnlp-main.130 · Microsoft UMA arXiv:2502.00943 · Flatiron PD-L1 aipo.2024.0043.*
