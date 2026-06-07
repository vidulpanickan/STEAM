# Dataset Format — Curation & Post-Processing

How to curate the assertion training data **once**, in a minimal model-agnostic form the LLM can
produce easily, and render it into the two training formats (ModernBERT encoder, Gemma 3 4B decoder)
**without re-labeling**.

Schema (frozen, from `SCHEMA_DESIGN.md`): `negated` (no/yes) · `certainty` (certain/possible) ·
`realis` (actual/hypothetical) · `experiencer` (self/family/other) · `temporality` (past/now/future).

---

## 1. Principle: curate one artifact, render two

The LLM produces one thing: the note text with **indexed entity markers inlined** plus an axis-label
block **keyed by marker index**. That single record renders to both models. You do **not** curate
per-entity, per-model, or with character offsets — the marker *is* the link between a position in the
text and its labels, which permanently resolves the "which occurrence?" problem with no
string-matching and no offsets.

Two deliberate simplifications for the LLM:
- **No entity type in the LLM output.** You already have types from your extraction step; making the
  model re-emit them is wasted effort and a chance to corrupt them. Pass type to the LLM as an
  *input hint*, and **join it back by index at render time**.
- **The LLM emits only `{text, entities}`.** `note_id`, `split`, and `source` are added by your
  pipeline around the call, not generated.

## 2. Clean steps (per note)

1. **Input to the LLM:** the original note text + the entity list from extraction (each entity's
   string and, as a hint, its type).
2. **LLM task:** inline a marker pair `[Ei] … [/Ei]` around **every occurrence** of each entity, and
   emit `{text, entities}` where each `Ei` carries its five axis values. (Marking convention in §3.)
3. **Validate the emission** (§9): stripping all markers from the returned `text` must reproduce the
   **original note text verbatim** (catches the LLM silently editing the note); markers are balanced
   and uniquely indexed; every marked index has a label block and vice-versa.
4. **Pipeline adds metadata:** attach `note_id`, assign `split` **by note**, set `source`
   (`real-llm` / `cf-rewrite` / `template`), and **join `type` by index** from your extraction data.
5. **Render** the encoder rows (§7) and the decoder messages (§8) from the canonical record.

## 3. What the LLM emits (minimal schema) + the marking rule

```json
{
  "text": "note text with [Ei] … [/Ei] markers inlined around each entity occurrence",
  "entities": {
    "E1": {"text": "...", "negated": "...", "certainty": "...", "realis": "...", "experiencer": "...", "temporality": "..."},
    "E2": { ... }
  }
}
```

**The marking rule — one marker per OCCURRENCE, not per entity string.** If the same term appears
more than once, each occurrence gets its **own index and its own label block** — never merged, even
when the labels are identical. This is the whole reason marking-at-curation works: the index points
at a *position*, so two mentions of the same words can carry different labels with zero ambiguity.
(The model's natural tendency is to collapse repeats into one entry — and the dropped mention is
often the one with the *different* label, so this rule is load-bearing. State it explicitly in the
generation prompt.)

The per-entity `"text"` field is redundant with the marked span but kept as a free self-consistency
check (validator confirms `Ei.text` equals what the `[Ei]` markers actually wrap).

## 4. The stored canonical record (after the pipeline step)

```json
{
  "note_id": "string (unique)",
  "split": "train | val | test",          // assigned BY NOTE, never by entity
  "source": "real-llm | cf-rewrite | template",
  "text": "... [Ei] … [/Ei] ...",
  "entities": {
    "E1": {"text": "...", "type": "Condition|Drug|Procedure|Measurement|Observation",  // type JOINED from extraction by index
           "negated": "...", "certainty": "...", "realis": "...", "experiencer": "...", "temporality": "..."}
  }
}
```

Axis values stay **canonical names**, never ints or verbalizers — those are produced at render time.

## 5. Shared label map (single source for both renders)

```python
LABEL2ID = {                       # encoder: name -> int
  "negated":     {"no": 0, "yes": 1},
  "certainty":   {"certain": 0, "possible": 1},
  "realis":      {"actual": 0, "hypothetical": 1},
  "experiencer": {"self": 0, "family": 1, "other": 2},
  "temporality": {"past": 0, "now": 1, "future": 2},
}
# decoder VERBALIZERS = the canonical names themselves (emit "possible", "hypothetical", …);
# confirm each is single-token in the target tokenizer before freezing.
```

Version-pin this file; both renders read from it, guaranteeing `certainty=possible` is the same class
to the encoder head and the decoder token.

---

## 6. Worked examples (LLM-output form — no type)

### Example 1 — ED differential (present-certain, present-possible, negated-certain)
```json
{
  "text": "39F on OCP, sudden pleuritic [E1] chest pain [/E1] after long flight. CXR clear. [E2] PE [/E2] concerning given tachycardia and travel history. [E3] Pleurisy [/E3] possible after recent viral illness. No [E4] leg swelling [/E4]. CTA ordered.",
  "entities": {
    "E1": {"text": "chest pain",  "negated": "no",  "certainty": "certain",  "realis": "actual", "experiencer": "self", "temporality": "now"},
    "E2": {"text": "PE",          "negated": "no",  "certainty": "possible", "realis": "actual", "experiencer": "self", "temporality": "now"},
    "E3": {"text": "Pleurisy",    "negated": "no",  "certainty": "possible", "realis": "actual", "experiencer": "self", "temporality": "now"},
    "E4": {"text": "leg swelling","negated": "yes", "certainty": "certain",  "realis": "actual", "experiencer": "self", "temporality": "now"}
  }
}
```
(`E2` is the abbreviation "PE" — the case string-matching would mis-locate; the marker fixes it.)

### Example 2 — history, family, plan, conditional (past, family, actual+future, hypothetical)
```json
{
  "text": "PMH significant for [E1] MI [/E1] in 2019. Mother had [E2] breast cancer [/E2] at age 45. Will start [E3] atorvastatin [/E3] today. If [E4] chest pain [/E4] recurs, return to the ED.",
  "entities": {
    "E1": {"text": "MI",            "negated": "no", "certainty": "certain", "realis": "actual",       "experiencer": "self",   "temporality": "past"},
    "E2": {"text": "breast cancer", "negated": "no", "certainty": "certain", "realis": "actual",       "experiencer": "family", "temporality": "past"},
    "E3": {"text": "atorvastatin",  "negated": "no", "certainty": "certain", "realis": "actual",       "experiencer": "self",   "temporality": "future"},
    "E4": {"text": "chest pain",    "negated": "no", "certainty": "certain", "realis": "hypothetical", "experiencer": "self",   "temporality": "future"}
  }
}
```
(`E3` "will start … today" is an **issued plan** → `actual`+`future`. `E4` "if … recurs" is irrealis → `hypothetical`.)

### Example 3 — ruled out, hedged-negative, denial, contact (the negation×certainty cross-product)
```json
{
  "text": "CT ruled out [E1] aortic dissection [/E1]. [E2] Pneumonia [/E2] unlikely given clear CXR. Patient denies [E3] fever [/E3]. Husband recently treated for [E4] influenza [/E4].",
  "entities": {
    "E1": {"text": "aortic dissection", "negated": "yes", "certainty": "certain",  "realis": "actual", "experiencer": "self",  "temporality": "now"},
    "E2": {"text": "Pneumonia",         "negated": "yes", "certainty": "possible", "realis": "actual", "experiencer": "self",  "temporality": "now"},
    "E3": {"text": "fever",             "negated": "yes", "certainty": "certain",  "realis": "actual", "experiencer": "self",  "temporality": "now"},
    "E4": {"text": "influenza",         "negated": "no",  "certainty": "certain",  "realis": "actual", "experiencer": "other", "temporality": "past"}
  }
}
```
(`E1` "ruled out" → `yes`+`certain`; `E2` "unlikely" → `yes`+`possible`, a hedged **negative**; `E4` "husband" → `other`.)

### Example 4 — the SAME entity twice with different labels (the repeat rule)
```json
{
  "text": "No [E1] chest pain [/E1] at rest, but reports [E2] chest pain [/E2] on exertion. [E3] Dyspnea [/E3] also noted.",
  "entities": {
    "E1": {"text": "chest pain", "negated": "yes", "certainty": "certain", "realis": "actual", "experiencer": "self", "temporality": "now"},
    "E2": {"text": "chest pain", "negated": "no",  "certainty": "certain", "realis": "actual", "experiencer": "self", "temporality": "now"},
    "E3": {"text": "Dyspnea",    "negated": "no",  "certainty": "certain", "realis": "actual", "experiencer": "self", "temporality": "now"}
  }
}
```
(Same words, **opposite** `negated`: `E1` denied at rest, `E2` affirmed on exertion. Two indices, two
label blocks — never merged.)

---

## 7. Post-processing → ModernBERT (encoder)

ModernBERT trains on **marked text + integer labels** — no prompt, no JSON. The standard
entity-anchored setup marks the **one target entity** per example and classifies it (R-BERT style:
insert special tokens around the target, pool their hidden states with `[CLS]`). Typed markers beat
plain ones by a few F1, so join the type into the marker at render.

**Render procedure (one example per entity / per marked occurrence):**
1. For each `Ei`, make one copy of `text` where **only `Ei`** is wrapped in a target marker
   `[E:<type>] … [/E:<type>]` (type joined from extraction); **strip the other indices** to plain
   text (their words remain as context).
2. Convert each axis name → int via `LABEL2ID`.
3. Emit `{text, negated, certainty, realis, experiencer, temporality}`.

**Rendered rows (from Example 1, `E1` and `E2`):**
```json
{"text": "39F on OCP, sudden pleuritic [E:Observation] chest pain [/E:Observation] after long flight. CXR clear. PE concerning given tachycardia and travel history. Pleurisy possible after recent viral illness. No leg swelling. CTA ordered.", "negated": 0, "certainty": 0, "realis": 0, "experiencer": 0, "temporality": 1}
{"text": "39F on OCP, sudden pleuritic chest pain after long flight. CXR clear. [E:Condition] PE [/E:Condition] concerning given tachycardia and travel history. Pleurisy possible after recent viral illness. No leg swelling. CTA ordered.", "negated": 0, "certainty": 1, "realis": 0, "experiencer": 0, "temporality": 1}
```
A repeat (Example 4) renders to **two separate rows**, each marking one occurrence — one with
`negated=1`, one with `negated=0`.

**Setup notes:**
- **Add markers as special tokens** (never split): `tokenizer.add_special_tokens({"additional_special_tokens": ["[E:Condition]","[/E:Condition]", …]})`,
  then `model.resize_token_embeddings(len(tokenizer))`. **Initialize the new marker embeddings to the
  mean of the existing embeddings** (rather than random) for faster convergence. Keep the type-pair
  set small; if you'd rather not reintroduce type at all, plain `[E] … [/E]` markers also work
  (marginally lower F1).
- **Model id** `answerdotai/ModernBERT-base`; 8,192-token context → feed the **whole note**. Tokenize
  `truncation=True, padding=True`; rename label column(s) to match the Trainer.
- **Two head layouts:** (A) five separate `AutoModelForSequenceClassification` models, each on one
  axis column (simplest); or (B) one shared encoder + five linear heads, summed cross-entropy (one
  forward pass). Rows are identical either way.
- **Empties:** a no-entity note → **zero encoder rows**; add explicit "no-target" negatives only if
  you want that signal.

## 8. Post-processing → Gemma 3 4B (decoder)

Gemma trains on **chat-format** text. **Keep all indexed markers inline** in the note (every entity,
repeats as distinct indices); the marker is the pointer, so **no entity list goes in the prompt** —
the prompt carries only the task and the allowed axis values. The model returns a **JSON array**, one
flat object per marked entity, in marker order. This is parity-safe because your extraction step
marks entities before the decoder sees them at inference too. (If instead your decoder will read
*unmarked* text at inference, strip the markers and pass an indexed entity list — but the marked form
is simpler and is what we recommend given your pipeline.)

**Use meaningful field names and values — never codes.** Each axis value is a word the model already
understands (`"possible"`, `"hypothetical"`), not an integer. Small models (1–12B) lean heavily on
the pretrained meaning of label tokens and cannot easily learn arbitrary code↔meaning mappings, so
`{"ax1":0,"ax2":1}` is slower to learn, more brittle, and gives no human-readable validation signal.
(Integers are correct for the *encoder* heads — a head emits a class index — but the *decoder*
generates tokens, which should be the verbalizer words. Don't import the encoder's encoding here.)
Each object also **echoes the entity span** (`"entity":"chest pain"`) for grounding and validation —
copying the span anchors the model's judgment to the actual words — plus an **`id`** that maps to the
inline marker. The `id`, not the span, is what stays unambiguous when a string repeats.

**Render procedure:**
1. Keep the indexed markers inline in `text` (optionally typed, `[E1:Condition]`, joined from
   extraction; plain `[E1]` is fine).
2. `user` message = task instruction + the allowed values per axis (no entity list).
3. `assistant` message = a JSON **array**, one object per marked entity in order, each
   `{"id", "entity", "negated", "certainty", "realis", "experiencer", "temporality"}`, meaningful values.

**Rendered example (from Example 1):**
```json
{"messages": [
  {"role": "user", "content": "For each entity marked [E1], [E2], … assign one value per axis.\nnegated: no|yes · certainty: certain|possible · realis: actual|hypothetical · experiencer: self|family|other · temporality: past|now|future\nReturn a JSON array, one object per marked entity, in order.\n\n39F on OCP, sudden pleuritic [E1] chest pain [/E1] after long flight. CXR clear. [E2] PE [/E2] concerning given tachycardia and travel history. [E3] Pleurisy [/E3] possible after recent viral illness. No [E4] leg swelling [/E4]. CTA ordered."},
  {"role": "assistant", "content": "[{\"id\":\"E1\",\"entity\":\"chest pain\",\"negated\":\"no\",\"certainty\":\"certain\",\"realis\":\"actual\",\"experiencer\":\"self\",\"temporality\":\"now\"},{\"id\":\"E2\",\"entity\":\"PE\",\"negated\":\"no\",\"certainty\":\"possible\",\"realis\":\"actual\",\"experiencer\":\"self\",\"temporality\":\"now\"},{\"id\":\"E3\",\"entity\":\"Pleurisy\",\"negated\":\"no\",\"certainty\":\"possible\",\"realis\":\"actual\",\"experiencer\":\"self\",\"temporality\":\"now\"},{\"id\":\"E4\",\"entity\":\"leg swelling\",\"negated\":\"yes\",\"certainty\":\"certain\",\"realis\":\"actual\",\"experiencer\":\"self\",\"temporality\":\"now\"}]"}
]}
```
A repeated string (Example 4) is simply two objects with different `id` (`E1`, `E2`) and possibly
different labels — the inline markers disambiguate, so no "mention 1/2" annotation is needed.

**Enforce the output shape (constrained decoding).** This is closed-vocabulary classification, the
easiest case for schema enforcement: define a JSON schema (array → object → `id` + `entity` string +
the five enum-constrained axis fields) and use constrained decoding (Outlines / XGrammar / the
provider's structured-output mode) so the model **cannot** emit a bad field name or an
out-of-vocabulary value. Fine-tuning already teaches the format; enforcement is cheap insurance, most
valuable at inference. **Validation:** the set of `id`s in the output must equal the set of marker
indices in `text` (no missing, extra, or hallucinated entities).

**Setup notes:**
- **Format with the chat template:** `tokenizer.apply_chat_template(messages, tokenize=False)`. Gemma
  uses `<start_of_turn>user … <end_of_turn>` / `<start_of_turn>model … <end_of_turn>`; the
  `SFTTrainer` adds its own `<bos>`, so don't add one.
- **No system role** in Gemma's template — put the instruction at the top of the first user message.
- **Train on the response only:** mask the prompt (`train_on_responses_only`,
  `instruction_part="<start_of_turn>user\n"`, `response_part="<start_of_turn>model\n"`).
- **Model id** `google/gemma-3-4b-it`; LoRA/QLoRA via TRL `SFTTrainer` + PEFT: start `r=16`,
  `lora_alpha=32`, target `q/k/v/o`+MLP, `lr≈2e-4`; QLoRA (4-bit) if VRAM-bound;
  `merge_and_unload()` before deploy.
- **Empties:** assistant content `[]`.
- **Hard-axis fallback (don't add up front):** forcing JSON *during reasoning* can cost ~10–15% on
  genuinely multi-step reasoning, so if the Stage-7 eval shows `negated`/`realis` lagging, switch
  *those* to two-step — a short free-text rationale emitted **before** the array — rather than
  complicating the default. For a fine-tuned closed-enum classifier the single-step form is usually
  fine.

---

## 9. Validation checks (after labeling AND after augmentation)

- **No silent note edit:** stripping all markers from `text` reproduces the original note verbatim
  (the LLM both marks and labels, so confirm it didn't paraphrase or truncate).
- **Marker integrity:** every `[Ei]` has a matching `[/Ei]`; indices unique within a note; every
  labeled index appears in `text`; every inlined marker has a label block; `Ei.text` matches the
  wrapped span.
- **Every occurrence marked:** if an entity string appears N times in the note, there are N indices
  for it (no merged repeats).
- **Markers survive counterfactual rewriting:** re-run marker integrity after any Stage-3 rewrite.
- **Per-split marginals:** split by note but balance by entity, so check per-axis-value proportions
  **within each split**.
- **Decoder output coverage:** the set of `id`s the decoder emits equals the marker-index set in the
  note (catches missing, extra, or hallucinated entities).
- **Label fidelity:** re-label a sample independently + NegEx on `negated`; for controlled rewrites,
  confirm only the target axis changed.

## 10. Authoritative sources

- **ModernBERT fine-tuning** (text + integer `labels`, `label2id`/`id2label`,
  `AutoModelForSequenceClassification`, 8,192-token context): HuggingFace ModernBERT fine-tuning
  guides (philschmid.de; argilla example).
- **Special-token mechanics** (`add_special_tokens` → `resize_token_embeddings`; special tokens never
  split; mean-init new embeddings): HuggingFace tokenizer docs and community guidance.
- **Entity markers / typed markers / marker pooling:** Wu & He 2019 (R-BERT — special tokens around
  the target, pool them); Soares et al. 2019 ("Matching the Blanks"); Zhou & Chen 2021 ("An Improved
  Baseline for Sentence-level Relation Extraction" — typed markers outperform masks, ~91% F1 on
  Re-TACRED).
- **Clinical assertion as classification:** CAN-BERT (MIMIC-III + i2b2); John Snow Labs BioBERT BFSC.
- **Gemma 3 fine-tuning** (`apply_chat_template`, `<start_of_turn>` turns, `SFTTrainer` adds `<bos>`,
  response-only masking, LoRA/QLoRA via TRL+PEFT): Gemma 3 fine-tuning guides and the model card.
- **Meaningful labels over codes:** verbalizers map labels to semantically meaningful words (Schick &
  Schütze, PET); small 1–12B models anchor on pretrained label semantics and can't easily relearn
  arbitrary mappings ("Semantic Anchors in In-Context Learning"); meaningful words beat random tokens
  in prompt/prefix tuning (SK-Tuning).
- **Structured outputs / constrained decoding:** schema enforcement guarantees valid fields and
  allowed values (OpenAI Structured Outputs, Aug 2024; Gemini `response_schema`; Outlines / XGrammar);
  keep the schema flat and reason-then-format on hard cases ("Let Me Speak Freely?", Tam et al. 2024).
