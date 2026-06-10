# SCHEMA_PRO — Expanded Per-Entity Clinical Schema (v2)

The full target schema for STEAM_PRO, folding the original 5 assertion axes together with the
research-backed additions (see `SCHEMA_EXPANSION.md` §8–9 for provenance). Two tiers per entity:
**axes** (closed-class, default + explicit cue) and **slots** (open-vocabulary spans, null unless
stated). `type` selects which slots apply.

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
    // conditional — medication ⇒ status only (course/severity absent; they're for problems)
    "status":      "stopped"     // started | stopped | increased | decreased | continued | unchanged
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

A **problem** entity instead carries `course` + `severity` (and no `status`):

```json
"E7": {
  "entity":"Pneumonia","text":"PNA","type":"problem",
  "axes":{ "negated":"no","certainty":"possible","realis":"actual","experiencer":"self",
           "temporality":"now","course":"improving","severity":"unspecified" },
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
| `problem` / `finding` | `course` · `severity` | `scale_value` |
| `procedure` | — *(done/not-done via `negated`+`temporality`)* | (universal only) |
| `other` | — | (universal only) |

## 4. AXES — values, defaults, decision rule

All axes keep the RULES.md contract: **start from default, flip ONLY on an explicit surface cue
that syntactically scopes this entity; no clinical inference.**

| axis | values | default | flip cue |
|---|---|---|---|
| `negated` | yes / no | **no** | negation operator (`no`, `denies`, `ruled out`, `without`) |
| `certainty` | certain / possible | **certain** | hedge word (`likely`, `possible`, `?`, `concerning for`) |
| `realis` | actual / hypothetical | **actual** | irrealis marker (`if`, `would`, contingent) |
| `experiencer` | self / family / other | **self** | attribution (blood relative→family; non-blood/template→other) |
| `temporality` | past / now / future | **now** | tense/time marker (`history of`→past; `will`/`plan`→future) |
| `status` | started / stopped / increased / decreased / continued / unchanged | **unchanged** | change verb (`started on`, `d/c'd`, `increased to`, `continue`) |
| `course` | improving / stable / worsening / resolving / resolved / unspecified | **unspecified** | trend word (`improving`, `worsening`, `resolved`) |
| `severity` | mild / moderate / severe / unspecified | **unspecified** | severity word (`mild`, `severe`, `massive`, `trace`) |

Notes:
- `status` is the entity's **change event** (CMED). It makes the old "discontinued → past" rule
  explicit: `discontinued` → `status=stopped`, `temporality=past`, `negated=no` (still not a
  denial). `declined/refused/not done` stays `negated=yes` (it never happened), `status=unchanged`.
- `course` and `severity` are independent of `status`: `resolving` → `course=resolving`,
  `status=unchanged`, still present (`negated=no`, `temporality=now`); only `resolved` →
  `course=resolved`, `temporality=past`.

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
- `indication` → `{span, ref}` — the reason; `ref` links to the problem entity id (`"E3"`) when one
  is marked, else null. (Relation, not just a span.)

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
            "temporality":"now","status":"continued","course":"unspecified","severity":"unspecified" },
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

Vital example — `HR: 89 (57 - 99) bpm`:

```json
"E9": {
  "entity":"Heart rate","text":"HR","type":"vital",
  "axes":{ "negated":"no","certainty":"certain","realis":"actual","experiencer":"self",
           "temporality":"now","status":"unchanged","course":"unspecified","severity":"unspecified" },
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
  experiencer,temporality}`). v2 nests them under `axes` and adds 3 axes + `type` + `slots`.
- Migration: lift the 5 flat keys into `axes`, set the 3 new axes to their defaults, set `type`
  from upstream (or `other`), set every slot to `null`. A v1 record becomes a valid v2 record with
  no information loss — only new fields, all at their defaults/null.
- **Encoder (Option A)** consumes `type` + `axes` only (8 softmax heads) — slots are out of its
  reach. **Generative (Option B)** emits the whole object. This is the cleanest split: ship the
  encoder for the 8 axes, run the generative model (or a span-tagger) for slots. See
  `SCHEMA_EXPANSION.md` §5.

## 8. Invariants (carried from RULES.md, enforced for every new field)

1. Default + explicit cue for every axis; `unspecified`/default is never inferred.
2. Null unless surface evidence for every slot; no dose/date/flag imputed from world knowledge.
3. Span-faithful: store the verbatim span; normalization is a separate, independently-auditable field.
4. Syntactic scope governs both axis flips and slot attachment (a dose/date attaches to the entity
   it grammatically modifies, not the nearest token).
5. `abnormal_flag` and any interpretive field are text-cue only — labeling a value from reference
   knowledge reintroduces the LLM-judgment failure mode the project was built to avoid.
