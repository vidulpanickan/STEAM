# SCHEMA_PRO — Flat Per-Entity Clinical Schema (v3, LoRA target)

The single source of truth for the STEAM_PRO schema. Designed as a **generative / LoRA fine-tune
target**: every entity emits **one flat JSON object with the exact same keys, in the same order** —
no per-type variation, no nesting, no lists, no cross-entity relations. A fixed frame + mostly
closed-vocabulary values is what a LoRA model learns to reproduce reliably; varying shape is what
makes it emit malformed output. Slot values are the **verbatim text span** copied from the note;
unit/date normalization is a separate deterministic pass, never the model's job.

Every design decision is referenced to its source standard in §9; out-of-scope choices and
citations are in §10–11.

---

## 1. Record shape (unchanged container)

```json
{
  "text_id": "text_00000",
  "origin": "real",
  "source": "labelling",
  "group": "real",
  "text": "... Last dose of Antibiotics:  [E1] Cefipime [/E1] - [**2177-3-15**] 02:48 AM ...",
  "entities": {
    "E1": { /* flat entity object — see §2 */ }
  }
}
```

## 2. Entity object (the one fixed shape — every entity, identical keys)

```json
{
  "id": "E1", "entity": "Cefipime",
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
  "route": "IV",
  "frequency": null,
  "date": "[**2177-3-15**] 02:48 AM",
  "body_site": null
}
```

16 keys, always present, always in this order. `id` + `entity` (the verbatim marked span) come from
the marker; the rest are predicted. Fields that don't apply to the entity hold their default
(`status=none`, `severity=unspecified`) or `null` (the span fields). **No key is ever omitted** —
that fixed frame is the whole point.

## 3. `type` and which fields it makes meaningful

`type` is now just an output field — it **no longer gates the shape** (all 16 keys are always
emitted). It is a closed set, collapsed small for reliability:

`type` ∈ `medication | lab | problem | procedure | other`
- vitals (HR, BP, SpO2) → `lab`; symptoms / exam findings → `problem`.

Which fields actually carry content per type (the rest stay at default/`null`):

| type | typically populated | always default/null |
|---|---|---|
| `medication` | `status`, `value`+`unit` (dose), `route`, `frequency`, `date` | `severity` |
| `lab` (incl. vital) | `value`+`unit` (result), `date` | `status`, `severity`, `route`, `frequency` |
| `problem` | `severity`, `value`+`unit` (stage, e.g. `IIIb`/`AJCC`), `body_site`, `date` | `status`, `route`, `frequency` |
| `procedure` | `body_site`, `date` | `status`, `severity`, `value`, `unit`, `route`, `frequency` |
| `other` | (universal axes only) | the rest |

This table is guidance for labelers, **not** a shape rule — the model always emits every key.

## 4. Fields — values, defaults, decision rule

The 7 closed-vocab axes keep the RULES.md contract: **start from default, flip ONLY on an explicit
surface cue that syntactically scopes this entity; no clinical inference.** The 6 span fields are
**`null` unless the value is explicitly written in the note**, and hold the **verbatim span**.

**Closed-vocabulary axes (always present):**

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

**Span fields (always present; verbatim span or `null`):**

| field | holds | example span |
|---|---|---|
| `value` | the entity's quantity — med dose amount, lab/vital result, problem stage | `40`, `14`, `109/50`, `IIIb` |
| `unit` | unit for `value` | `mg`, `mcg/Kg/min`, `K/uL`, `mmHg` |
| `route` | medication route | `PO`, `IV`, `topical` |
| `frequency` | medication sig | `BID`, `q6h`, `PRN` |
| `date` | date/time tied to the entity | `[**2177-3-15**]`, `POD 2` |
| `body_site` | anatomical location | `RLL`, `left heel` |

Notes:
- `status` is the medication **change action** (CMED). Default `none` covers both an explicitly
  *continued* med (`continue home meds` → `none`) and no mention. Makes the old "discontinued → past"
  rule explicit: `discontinued` → `status=stopped`, `temporality=past`, `negated=no`.
  `declined/refused/not done` → `negated=yes`, `status=none`.
- **No `course` axis.** Trend/resolution rides on `temporality`+`negated`: `resolved` →
  `temporality=past`; `resolving`/`improving`/`worsening` → `temporality=now` (still present).
- **Compound values stay as one raw span** (`value="109/50"`, `value="7.37/36/115"`) — we dropped the
  per-component list to keep the shape flat; splitting BP/ABG is a downstream parse, not the model's.
- **`value`+`unit` is generalized:** dose for meds, result for labs/vitals, stage for problems —
  same two fields, so the model learns one pattern instead of per-type slots.

## 5. Worked examples (all identical 16-key shape)

Medication — `Infusions:  [E2] Norepinephrine [/E2] - 0.03 mcg/Kg/min`:

```json
{
  "id":"E2","entity":"Norepinephrine","type":"medication",
  "negated":"no","certainty":"certain","realis":"actual","experiencer":"self","temporality":"now",
  "status":"none","severity":"unspecified",
  "value":"0.03","unit":"mcg/Kg/min","route":"IV","frequency":null,
  "date":null,"body_site":null
}
```

Lab/vital — `HR: 89 (57 - 99) bpm`:

```json
{
  "id":"E9","entity":"HR","type":"lab",
  "negated":"no","certainty":"certain","realis":"actual","experiencer":"self","temporality":"now",
  "status":"none","severity":"unspecified",
  "value":"89","unit":"bpm","route":null,"frequency":null,
  "date":null,"body_site":null
}
```

Problem — `... concerning for RLL [E7] pneumonia [/E7], improving`:

```json
{
  "id":"E7","entity":"pneumonia","type":"problem",
  "negated":"no","certainty":"possible","realis":"actual","experiencer":"self","temporality":"now",
  "status":"none","severity":"unspecified",
  "value":null,"unit":null,"route":null,"frequency":null,
  "date":null,"body_site":"RLL"
}
```

(`certainty=possible` from "concerning for"; `temporality=now` carries "improving" — no course axis.)

## 6. Generation format (how these objects are served to the model)

Per `build_chat_dataset.py`, one training example per note:
- **system** = `FINAL_TRAINING_DATA/system_prompt.txt` (the schema contract).
- **user** = `{"text_id": ..., "text": <note with all [Ei] … [/Ei] markers>}`.
- **assistant** = a JSON **array** of the §2 objects, one per entity, **in marker order**.

So the only structure the model produces is a list of identical flat objects — no nesting beyond the
list, fixed keys throughout.

## 7. Backward compatibility with the current data

- Today's `training_data_all.jsonl` stores, per entity, `{id?, entity, negated, certainty, realis,
  experiencer, temporality}` (5 axes only). v3 adds 9 keys: `type`, `status`, `severity`, `value`,
  `unit`, `route`, `frequency`, `date`, `body_site`.
- Migration of an existing record: keep the 5 axis values, add `type` (from upstream; default
  `other`), `status=none`, `severity=unspecified`, and the 6 span fields `=null`. A v1 entity
  becomes a valid v3 object with no information loss — only new fields, all at defaults/null. The new
  fields then get **labeled** as a separate pass (they don't exist in the data yet).
- **Primary path is generative + LoRA** (this schema is its target). An encoder can still consume the
  8 closed-vocab fields (`type` + 7 axes) as classification heads; the 6 span fields require the
  generative model or a span-tagger.

## 8. Invariants (carried from RULES.md, enforced for every field)

1. Default + explicit cue for every axis; a default is never inferred.
2. `null` unless the value is written in the text for every span field; no dose/date imputed from
   world knowledge.
3. Span-faithful: span fields hold the **verbatim** text; normalization (UCUM, ISO date, numeric
   parse) is a separate downstream pass, not in the label.
4. Syntactic scope governs both axis flips and span attachment (a dose/date attaches to the entity
   it grammatically modifies, not the nearest token).
5. No interpretive fields computed from reference knowledge — that was why `abnormal_flag` (high/low
   from a reference range) was dropped: it reintroduces the LLM-judgment failure mode the project
   avoids.

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
| `route` | verbatim span | null | i2b2-2009 mode; FHIR Dosage.route | — |
| `frequency` | verbatim span | null | i2b2-2009/MedEx frequency; FHIR Dosage.timing | — |
| `date` | verbatim span | null | i2b2-2012 TIMEX3 | normalization downstream (MIMIC dates shifted) |
| `body_site` | verbatim span | null | cTAKES body location; FHIR bodySite | — |

**Normalization decisions (v3, for the LoRA target):**
- **Flattened** — removed the `axes`/`slots` two-tier nesting and the `{span, normalized}` objects;
  one flat object, span-only values.
- **De-varied** — `status`/`severity` made always-present (not type-conditional) so the shape is
  constant; `type` demoted from shape-gate to plain field.
- **Generalized** — med `dose` and lab `value` merged into one `value`+`unit` pair (+ problem stage).
- **Dropped** (see §10) — `indication` relation, `components[]` list, `reference_range`, `specimen`,
  `form`, `abnormal_flag`, `duration`, `scale_value`, and the earlier-evaluated `course` axis.

## 10. Out of scope / dropped (and why)

**Dropped to keep the LoRA target flat & fixed** (not because they're useless — they re-enter only
if a later eval shows they're worth the shape cost):
- `indication` — a cross-entity relation (drug→problem); references between entities are the most
  error-prone thing a generative model emits.
- `components[]` — a variable-length list (BP/ABG parts); kept instead as one raw `value` span.
- `reference_range`, `specimen`, `form`, `duration`, `scale_value` — long-tail slots; `scale_value`
  folds into `value`+`unit`.
- `abnormal_flag` — interpretive (high/low from a range) → violates invariant §8.5.
- `{span, normalized}` nesting — normalization moved to a deterministic downstream pass.

**Out of scope entirely** (specialty-registry universe, separate profile): oncology biomarkers
(PD-L1 CPS/TPS), genomic mutation status, tumor response/progression dates, performance status as
its own pipeline, SDoH, device attributes, demographics. (Ordinal scales clinicians do chart —
ECOG/NYHA/GCS/Child-Pugh — degrade into `value`+`unit` when written, e.g. `value="II"`, `unit="NYHA"`.)

## 11. Evidence base & sources

- **Medication signature set** (name/dose/route/frequency/duration/reason): i2b2-2009, MedEx,
  n2c2-2018, JSL posology, FHIR Dosage — the most universal attribute cluster. v3 keeps the highest-
  value subset (`value`/`unit`/`route`/`frequency`) as flat span fields.
- **Medication change events** (`status`): CMED 2021/2022 Action {Start, Stop, Increase, Decrease,
  UniqueDose, Unknown} — the analogue of the clinician "med changes" need.
- **Assertion cluster** (the 5 universal axes): i2b2-2010 assertion, i2b2-2012 polarity+modality,
  cTAKES, medspaCy ConText, OMOP `term_modifiers`. Negation + certainty appear in every system.
- **`severity` / `body_site`**: cTAKES nine-attribute set + FHIR Condition.severity/bodySite.
  (`course`, cTAKES-only with no second source, was dropped.)
- **Lab/Observation `value`/`unit`**: FHIR Observation.value[x]; NER shared tasks treat labs only as
  bare "test" concepts, which is why the value/unit fields are needed.
- **Clinician information-needs**: Assessment & Plan dominates chart review (Brown 2014 eye-tracking
  67%; Clarke 2014 needed by 92% of PCPs); discharge studies (Chatterton 2023; Sorita 2021; JGIM
  2025) rank med changes + dates highly — motivating `status`, `value`, `date`.
- **Recent LLM-IE (2023-26)**: Agrawal EMNLP-2022 (few-shot med-attribute + med-status schema);
  Microsoft UMA 2025; Flatiron progression+date / PD-L1. Unanimous caveat: a raw span is enough as
  the *label*; normalization (LOINC/units/RxNorm/dates) is a separate deterministic layer — exactly
  the span-only design here. Structured-output reliability scales with a **fixed, flat schema**.

*Key sources: i2b2-2009 PMC2995676 · MedEx PMC2995636 · n2c2-2018 PMC7489085 · CMED PMC10529825 ·
i2b2-2010 PMC3168320 · i2b2-2012 PMC3756273 · cTAKES Default Clinical Pipeline · FHIR
Observation/Condition/Dosage (build.fhir.org) · OMOP NOTE_NLP · Brown 2014 PMC4081746 · Clarke 2014
PMC3974234 · Chatterton 2023 PMC11169121 · Sorita 2021 PMID 28885382 · Agrawal 2022
aclanthology 2022.emnlp-main.130 · Microsoft UMA arXiv:2502.00943 · Flatiron PD-L1 aipo.2024.0043.*
