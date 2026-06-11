# SCHEMA_PRO — Expanded Per-Entity Clinical Schema (v2)

The single source of truth for the STEAM_PRO schema — the original 5 assertion axes folded together
with the research-backed additions. Two tiers per entity: **axes** (closed-class, default + explicit
cue) and **slots** (open-vocabulary spans, null unless stated). `type` selects which axes & slots
apply. Every design decision is referenced to its source standard in §9; out-of-scope choices and
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
    "E1": { /* entity object — see §2 */ }
  }
}
```

## 2. Entity object

Axes come in two groups: **5 universal** (every entity) + **conditional** axes that travel with the
`type` (just like slots). Only the keys for this entity's type are present.

```json
{
  "entity": "Cefepime",          // normalized concept name (upstream)
  "text":   "Cefipime",          // verbatim marked span
  "type":   "medication",        // see §3 — selects the conditional axes AND the slot set

  "axes": {
    // universal — always present, every entity
    "negated":     "no",         // yes | no
    "certainty":   "certain",    // certain | possible
    "realis":      "actual",     // actual | hypothetical
    "experiencer": "self",       // self | family | other
    "temporality": "past",       // past | now | future
    // conditional — medication ⇒ status only (severity absent; it's for problems)
    "status":      "stopped"     // started | stopped | increased | decreased | none
  },

  "slots": {                     // present keys depend on `type`; value is {…}|null, null = not stated
    "date":      {"span": "[**2177-3-15**] 02:48 AM", "normalized": null},  // ⚠ MIMIC dates shifted
    "duration":  null,
    "body_site": null,
    "dose":      null,
    "route":     {"span": "Antibiotics", "normalized": "IV"},
    "frequency": null,
    "form":      null,
    "indication":null
  }
}
```

A **problem** entity instead carries `severity` (and no `status`):

```json
"E7": {
  "entity":"Pneumonia","text":"PNA","type":"problem",
  "axes":{ "negated":"no","certainty":"possible","realis":"actual","experiencer":"self",
           "temporality":"now","severity":"unspecified" },
  "slots":{ "date":null,"duration":null,
            "body_site":{"span":"RLL","normalized":"right lower lobe","laterality":"right"},
            "scale_value":null }
}
```

## 3. Entity types → which axes & slots are emitted

`type` ∈ `medication | lab | vital | problem | finding | procedure | other`. The **5 universal
axes** and **universal slots** (`date`, `duration`, `body_site`) apply to all. Everything else is
conditional — an entity only carries the keys for its type (others simply absent).

| type | conditional axes | type-conditional slots |
|---|---|---|
| `medication` | `status` | `dose` · `route` · `frequency` · `form` · `indication` |
| `lab` / `vital` | — | `value` · `unit` · `reference_range` · `abnormal_flag` · `specimen` · `components[]` |
| `problem` / `finding` | `severity` | `scale_value` |
| `procedure` | — *(done/not-done via `negated`+`temporality`)* | (universal only) |
| `other` | — | (universal only) |

## 4. AXES — values, defaults, decision rule

All axes keep the RULES.md contract: **start from default, flip ONLY on an explicit surface cue
that syntactically scopes this entity; no clinical inference.**

**Universal (every entity):**

| axis | values | default | flip cue |
|---|---|---|---|
| `negated` | yes / no | **no** | negation operator (`no`, `denies`, `ruled out`, `without`) |
| `certainty` | certain / possible | **certain** | hedge word (`likely`, `possible`, `?`, `concerning for`) |
| `realis` | actual / hypothetical | **actual** | irrealis marker (`if`, `would`, contingent) |
| `experiencer` | self / family / other | **self** | attribution (blood relative→family; non-blood/template→other) |
| `temporality` | past / now / future | **now** | tense/time marker (`history of`→past; `will`/`plan`→future) |

**Conditional (only on the owning type — see §3):**

| axis | owner type | values | default | flip cue |
|---|---|---|---|---|
| `status` | medication | started / stopped / increased / decreased / none | **none** | change verb (`started on`, `d/c'd`, `increased to`, `decreased`) |
| `severity` | problem / finding | mild / moderate / severe / unspecified | **unspecified** | severity word (`mild`, `severe`, `massive`, `trace`) |

Notes:
- `status` is the medication **change action** (CMED Action set, minus its rarely-needed
  UniqueDose/OtherChange). Default **`none`** = no change action stated, which covers an
  explicitly *continued* med (`continue home meds` → `none`) as well as no mention. It makes the
  old "discontinued → past" rule explicit: `discontinued` → `status=stopped`, `temporality=past`,
  `negated=no` (still not a denial). `declined/refused/not done` stays `negated=yes` (it never
  happened), `status=none`.
- **No `course` axis.** Trend/resolution is carried by `temporality` + `negated`: `resolved` →
  `temporality=past` (`negated=no`); `resolving` / `improving` / `worsening` → `temporality=now`
  (still present). `severity` is independent of `status`/`temporality` and scopes the finding's
  intensity only.

## 5. SLOTS — definition & shape

Every slot is `null` unless explicitly stated in the text. Each carries the **verbatim `span`**;
normalization is a **separate field** so it can be audited/corrected independently (and so an
unlearnable normalization — e.g. shifted MIMIC dates — degrades to `null` without losing the span).

### Universal
- `date` → `{span, normalized}` — absolute (`[**2177-3-14**]`) or relative (`POD 2`, `on admission`).
  ⚠ MIMIC dates are de-identified/shifted: extract the span, leave `normalized` null.
- `duration` → `{span, normalized}` — `x 7 days`, `for 3 weeks`.
- `body_site` → `{span, normalized, laterality}` — anatomical location; `laterality` ∈
  `left|right|bilateral|null`. Applies to findings/problems/procedures, not just procedures.

### Medication
- `dose` → `{span, value, unit}` — `40 mg` → `{40, "mg"}`; `0.03 mcg/Kg/min` → `{0.03, "mcg/kg/min"}`.
- `route` → `{span, normalized}` — PO / IV / SC / topical / inhaled.
- `frequency` → `{span, normalized}` — BID, q6h, PRN, daily (the medication sig, not symptom freq).
- `form` → `{span, normalized}` — tablet, patch, infusion.
- `indication` → **a relation, not a closed slot.** The reason a drug is given is open-ended and
  often a whole clause (`"for volume overload"`, `"to prevent DVT given recent hip surgery"`), so it
  is **not** normalized to a vocabulary. Shape: `{ref, span}` where `ref` is the **problem entity
  id** the drug treats (`"E5"`) when that problem is separately marked — the n2c2-2018 Reason→Drug
  relation — and `span` holds the verbatim reason text. `ref` is primary; `span` is the fallback
  when no problem entity is co-marked. May be a **list** if a drug has multiple indications.

### Lab / vital
- `value` → `{span, parsed}` — numeric or coded result.
- `unit` → `{span, normalized}` — UCUM where possible (`mEq/L`, `mcg/kg/min`).
- `reference_range` → `{span, low, high}` — only if stated in text.
- `abnormal_flag` → `high | low | normal | abnormal | null` — **text-cue only** (`elevated`, `H`/`L`
  flag, `wnl`); never computed from reference knowledge.
- `specimen` → `{span, normalized}` — blood / urine / CSF, if stated.
- `components[]` → list of `{label, value, unit}` — for compound values: BP `109/50` →
  `[{systolic,109,mmHg},{diastolic,50,mmHg}]`; ABG `7.37/36/115/22/-3` → pH/pCO2/pO2/HCO3/BE.

### Problem / finding
- `scale_value` → `{scale, value}` — ordinal value on a **named** scale: cancer `{TNM/Stage, "IIIb"}`,
  `{CKD, "3"}`, `{ECOG, "2"}`, `{NYHA, "III"}`, `{GCS, "14"}`, `{Child-Pugh, "B"}`.

## 6. Worked example (real record from `training_data_all.jsonl`)

Source line: `Infusions:  [E2] Norepinephrine [/E2] - 0.03 mcg/Kg/min` and
`Last dose of Antibiotics:  [E1] Cefipime [/E1] - [**2177-3-15**] 02:48 AM`.

```json
"E2": {
  "entity": "Norepinephrine", "text": "Norepinephrine", "type": "medication",
  "axes": { "negated":"no","certainty":"certain","realis":"actual","experiencer":"self",
            "temporality":"now","status":"none" },
  "slots": {
    "date":null, "duration":null, "body_site":null,
    "dose":{"span":"0.03 mcg/Kg/min","value":0.03,"unit":"mcg/kg/min"},
    "route":{"span":"Infusions","normalized":"IV"},
    "frequency":{"span":"infusion","normalized":"continuous"},
    "form":{"span":"Infusions","normalized":"infusion"},
    "indication":null
  }
}
```

Vital example — `HR: 89 (57 - 99) bpm` (no conditional axes; vitals carry only the universal 5):

```json
"E9": {
  "entity":"Heart rate","text":"HR","type":"vital",
  "axes":{ "negated":"no","certainty":"certain","realis":"actual","experiencer":"self",
           "temporality":"now" },
  "slots":{
    "date":null,"duration":null,"body_site":null,
    "value":{"span":"89","parsed":89}, "unit":{"span":"bpm","normalized":"/min"},
    "reference_range":{"span":"(57 - 99)","low":57,"high":99},
    "abnormal_flag":null, "specimen":null, "components":[]
  }
}
```

## 7. Backward compatibility with v1

- The v1 file stores the 5 axes **flat** on the entity (`{entity,text,negated,certainty,realis,
  experiencer,temporality}`). v2 nests them under `axes` and adds `type`, the conditional axes, and
  `slots`.
- Migration: lift the 5 flat keys into `axes`, set `type` from upstream (or `other`), add the
  conditional axes for that type at their defaults, set every slot to `null`. A v1 record becomes a
  valid v2 record with no information loss — only new fields, all at defaults/null.
- **Encoder (Option A)**: 5 always-on heads (universal axes) + per-type conditional heads
  (`status` for meds, `severity` for problems/findings) trained with a **type-masked loss** — a
  head is only supervised on entities of its owning type. Slots and relations (`indication`, components) are out of the
  encoder's reach. **Generative (Option B)** emits the whole object in one pass — including the
  conditional axes and the `indication` relation. This is the cleanest split: ship the encoder for
  the axes, run the generative model (or a span-tagger + relation layer) for slots/relations.

## 8. Invariants (carried from RULES.md, enforced for every new field)

1. Default + explicit cue for every axis; `unspecified`/default is never inferred.
2. Null unless surface evidence for every slot; no dose/date/flag imputed from world knowledge.
3. Span-faithful: store the verbatim span; normalization is a separate, independently-auditable field.
4. Syntactic scope governs both axis flips and slot attachment (a dose/date attaches to the entity
   it grammatically modifies, not the nearest token).
5. `abnormal_flag` and any interpretive field are text-cue only — labeling a value from reference
   knowledge reintroduces the LLM-judgment failure mode the project was built to avoid.

## 9. Axis decision log (each value set → its source)

| axis | final values | default | source standard | note / overlap removed |
|---|---|---|---|---|
| `negated` | yes / no | no | i2b2-2010 assertion; NegEx | — |
| `certainty` | certain / possible | certain | i2b2-2010 (`possible`); cTAKES uncertainty | — |
| `realis` | actual / hypothetical | actual | i2b2-2010 conditional/hypothetical; i2b2-2012 modality | — |
| `experiencer` | self / family / other | self | i2b2-2010 not-patient; cTAKES subject | — |
| `temporality` | past / now / future | now | i2b2-2012 temporal; CMED temporality | absorbs resolved→past, resolving→now |
| `status` (med) | started / stopped / increased / decreased / **none** | none | CMED Action set | dropped `continued`+`unchanged` → single `none` |
| `severity` (problem/finding) | mild / moderate / severe / unspecified | unspecified | cTAKES severity; FHIR Condition.severity | — |
| ~~`course`~~ | — **dropped** — | — | cTAKES-only (not in FHIR/CMED/i2b2/n2c2) | overlapped `temporality`; removed |

Two label-clarity fixes recorded here: **(a)** `status` collapsed the confusable `continued`/
`unchanged` pair into one `none` default (CMED has no "continued" — a continued med is a *no-change*
event); **(b)** `course` was evaluated and **dropped** — it is supported only by cTAKES and its
informative values (`resolved`/`resolving`) already fall out of `temporality`, so it added a
confusing axis with no cross-standard backing.

## 10. Out of scope (deliberately excluded)

Kept out to avoid over-generalizing a general clinical schema into specialty registries:
oncology biomarkers (PD-L1 CPS/TPS), genomic mutation status, tumor response/progression dates,
performance status as its own pipeline, SDoH/social determinants, device attributes, demographics.
These are specialty-registry concerns (Flatiron/Tempus/Microsoft-UMA) with their own normalization
vocabularies (ICD-O, MedDRA) and whole-history reasoning; they belong in a separate oncology/
registry profile, not here. (Ordinal severity scales clinicians *do* chart in general notes —
ECOG, NYHA, GCS, Child-Pugh — are captured generically by the `scale_value` slot, §5.)

## 11. Evidence base & sources

- **Medication signature set** (name/strength/form/dose/route/frequency/duration/reason) is the most
  universal attribute cluster: i2b2-2009, MedEx, n2c2-2018 (9 concepts + drug-anchored relations),
  JSL posology, FHIR Dosage.
- **Medication change events** (`status`): CMED 2021/2022 Action {Start, Stop, Increase, Decrease,
  UniqueDose, OtherChange, Unknown} + negation/temporality/certainty/actor — the direct analogue of
  the clinician "med changes" need.
- **Assertion/context cluster** (the universal 5): i2b2-2010 assertion (present/absent/possible/
  conditional/hypothetical/not-patient), i2b2-2012 polarity+modality, cTAKES, medspaCy ConText,
  OMOP `term_modifiers`. Negation + certainty appear in *every* surveyed system.
- **`severity` / `body_site`**: cTAKES nine-attribute set + FHIR Condition.severity/bodySite; JSL.
  (cTAKES's `course` was the one attribute with no second source — hence dropped.)
- **Lab/Observation semantics** (value/unit/referenceRange/interpretation/specimen/component):
  modeled almost only by FHIR Observation; NER shared tasks treat labs as bare "test" concepts.
- **Clinician information-needs**: Assessment & Plan dominates chart review (Brown 2014 eye-tracking
  67% of reading time; Clarke 2014 needed by 92% of PCPs). Discharge-summary studies (Chatterton
  2023; Sorita 2021; JGIM 2025) rank med changes *with reasons + stop dates*, pending results/
  follow-up *with dates*, and hospital course at the top — motivating `status`, `indication`,
  `date`, `duration`.
- **Recent LLM-IE (2023-26)**: Agrawal EMNLP-2022 (few-shot med-attribute + med-status schema);
  Microsoft UMA 2025 (short- vs long-context attribute-difficulty split); Flatiron progression+date
  / PD-L1 7-field. Unanimous caveat: a raw span is insufficient — a separate deterministic
  normalization layer (LOINC/units/RxNorm/dates) is required, which is why every slot keeps span
  and normalized as distinct fields.

*Key sources: i2b2-2009 PMC2995676 · MedEx PMC2995636 · n2c2-2018 PMC7489085 · CMED PMC10529825 ·
i2b2-2010 PMC3168320 · i2b2-2012 PMC3756273 · cTAKES Default Clinical Pipeline · FHIR
Observation/Condition/Dosage (build.fhir.org) · OMOP NOTE_NLP · Brown 2014 PMC4081746 · Clarke 2014
PMC3974234 · Chatterton 2023 PMC11169121 · Sorita 2021 PMID 28885382 · Agrawal 2022
aclanthology 2022.emnlp-main.130 · Microsoft UMA arXiv:2502.00943 · Flatiron PD-L1 aipo.2024.0043.*
