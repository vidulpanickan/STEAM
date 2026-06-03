# Inference Pipeline — STEAM Demo Dataset → MedGemma 1.5 4B (Q8_0, llama.cpp)

End-to-end documentation for running the fine-tuned MedGemma 1.5 4B clinical-entity classifier on a clinical-notes DataFrame using llama.cpp. Three small scripts + the model checkpoint do the whole pipeline.

This folder (`tools/inference/`) is self-contained: it holds the three pipeline scripts plus the bundled data artifacts (`all_terms.csv`, `all_terms_typed.csv`, the `inference_inputs.*` set, and the `model_responses.top*.jsonl` samples). The one-time training-data construction and fine-tuning scripts live separately under `tools/training/` and are not needed to run inference. All commands below are written to be run from the repository root.

## Running the pipeline (quick reference)

The pipeline is **three scripts run in sequence**: build inputs → run the model → merge results. Everything else here is detail.

| # | Script | Consumes | Produces |
|---|--------|----------|----------|
| 1 | `build_inference_input.py` | your clinical-notes CSV + the term catalog (`all_terms_typed.csv`) | `inference_inputs.jsonl` (one window per line, model input) + `inference_inputs.tsv` (one row per window×entity, axis columns empty) |
| 2 | `run_llamacpp_inference.py` | `inference_inputs.jsonl` + the `medgemma-1.5-4b-it.Q8_0.gguf` checkpoint | `model_responses.jsonl` (the model's per-window JSON-array predictions) |
| 3 | `merge_inference_outputs.py` | `inference_inputs.tsv` + `model_responses.jsonl` | `predictions.tsv` (the TSV with all axis columns filled in + a `status` column) |

```bash
# 1. Build windows from notes + the typed term catalog
python tools/inference/build_inference_input.py \
  --notes     ct_reports.csv \
  --terms     tools/inference/all_terms_typed.csv \
  --out       inference_inputs.jsonl \
  --out-table inference_inputs.tsv

# 2. Run the fine-tuned model on every window (one llama-cli call per window)
python tools/inference/run_llamacpp_inference.py \
  --model  /path/to/medgemma-1.5-4b-it.Q8_0.gguf \
  --inputs inference_inputs.jsonl \
  --out    model_responses.jsonl

# 3. Merge the per-window predictions back into the per-entity table
python tools/inference/merge_inference_outputs.py \
  --table     inference_inputs.tsv \
  --responses model_responses.jsonl \
  --out       predictions.tsv
```

**What each stage does, in one line:**
- **Stage 1 — build:** scans each note for catalog terms, clusters nearby hits into context windows (≤1500 chars), and emits one model-ready record per window plus an analysis-ready TSV stub. The system prompt is embedded byte-for-byte from the fine-tune.
- **Stage 2 — run:** sends each window through `llama-cli` against the Q8_0 GGUF at `--temp 0.0` (deterministic), capturing the assistant turn — a JSON array assigning the SCHEMA3 axes (assertion / certainty / temporality / subject / type-specific 5th) to every entity in the window.
- **Stage 3 — merge:** joins responses back by `(patient_id, note_id, window_start, window_end)`, fills the axis columns per entity, and tags each row `ok` / `parse_error` / `entity_missing` / `no_response`. The result, `predictions.tsv`, is ready for pandas/SQL.

The sections below document each stage, its flags, the windowing strategy, performance tuning, and known behaviors in full.

## Toolchain assumed

- **Model:** `medgemma-1.5-4b-it.Q8_0.gguf` — MedGemma 1.5 4B Instruct, 8-bit GGUF quantization. Q8_0 is essentially lossless (≤0.5% perplexity drift vs fp16), so quantization-related quality concerns are negligible compared to Q4 variants.
- **Runtime:** `llama-cli` (and optionally `llama-server`) from a standard llama.cpp build. Metal/CUDA/CPU all supported via the same CLI.
- **Python 3.10+** with `pandas`. No other external dependencies.

## Input file you provide

A single CSV (or TSV / parquet) with **exactly these columns**:

```
empi   EPIC_PMRN   MRN_Type   MRN   Report_Number   Report_Date_Time
Report_Description   Report_Status   Report_Type   report
```

- `empi` → patient ID (used).
- `Report_Number` → note ID (used).
- `report` → the clinical note body (used).
- Everything else is loaded but unused — pass it through unchanged.

All columns load as strings (no float coercion of `empi` / `MRN` / `Report_Number`). The script auto-detects format from extension: `.csv` → comma-delimited, `.tsv`/`.tab` → tab-delimited, `.parquet` → parquet. If your file is tab-delimited but named `.csv`, rename to `.tsv` or load it yourself with `pd.read_csv(..., sep="\t", dtype=str)` and call `build_inference_records()` directly.

## End-to-end workflow

Four steps. Each step is a single CLI invocation. The defaults are already wired for the EPIC schema above — no column-override flags needed.

```bash
# 1. Build inference inputs (JSONL for model + long-form TSV for analysis)
python tools/inference/build_inference_input.py \
  --notes     ct_reports.csv \
  --terms     tools/inference/all_terms_typed.csv \
  --out       inference_inputs.jsonl \
  --out-table inference_inputs.tsv

# 2. Run the model on every window
python tools/inference/run_llamacpp_inference.py \
  --model  /path/to/medgemma-1.5-4b-it.Q8_0.gguf \
  --inputs inference_inputs.jsonl \
  --out    model_responses.jsonl

# 3. Merge model responses into the analysis table
python tools/inference/merge_inference_outputs.py \
  --table     inference_inputs.tsv \
  --responses model_responses.jsonl \
  --out       predictions.tsv

# 4. Analyze
#   predictions.tsv is now ready for pandas/SQL: one row per (window, entity)
#   with all axis predictions filled in plus a 'status' column.
```

The rest of this document explains each stage in detail.

## Context

The MedGemma 1.5 4B fine-tune was trained on records of the form `{messages: [system, user, assistant]}` where:
- `system` = a fixed clinical-entity-classifier prompt describing the SCHEMA3 axes.
- `user` = a JSON string `{"entities": [...], "sentence": "<window>"}`.
- `assistant` = a JSON array assigning axis values to each entity.

To run inference on fresh notes (EPIC export of chest CT reports, say), we produce the same `messages[0..1]` structure, let the model generate the assistant turn, then expand the per-window assistant turns into a per-entity tabular file for downstream analysis.

Related files:
- `tools/inference/build_inference_input.py` — stage 1 (build).
- `tools/inference/run_llamacpp_inference.py` — stage 2 (run inference via llama-cli).
- `tools/inference/merge_inference_outputs.py` — stage 3 (merge responses into the analysis table).
- `tools/inference/all_terms_typed.csv` — 107 chest-CT terms, all typed `Condition`.
- `../../DATASET_CURATION.md` — how the training dataset was built (separate concern, retained for reference).
- `../../SCHEMA3.md` — axis definitions and cue-word lists.

## Stage 1: Build inputs (`build_inference_input.py`)

### Inputs

**Notes file** — your EPIC CSV / TSV / parquet with the columns listed in [Input file you provide](#input-file-you-provide). Column mapping (built-in defaults):

| Source column         | Used as       | Notes                                            |
|-----------------------|---------------|--------------------------------------------------|
| `empi`                | `patient_id`  | Loaded as string to preserve leading zeros.       |
| `EPIC_PMRN`           | (unused)      | Pass-through; ignored.                            |
| `MRN_Type`            | (unused)      | "                                                 |
| `MRN`                 | (unused)      | "                                                 |
| `Report_Number`       | `note_id`     | Loaded as string.                                 |
| `Report_Date_Time`    | (unused)      | "                                                 |
| `Report_Description`  | (unused)      | "                                                 |
| `Report_Status`       | (unused)      | "                                                 |
| `Report_Type`         | (unused)      | "                                                 |
| `report`              | `text`        | The clinical note body to scan.                    |

All columns load with `dtype=str` (CSV/TSV) or `.astype("string")` (parquet) so numeric-looking IDs don't get silently coerced. Empty strings become `NaN` (skipped); explicit `"NaN"` strings in the data are preserved.

If the columns are renamed in some future export, override with `--text-col`, `--patient-id-col`, `--note-id-col`.

**Term catalog** — CSV/TSV/parquet with two columns: `term` and `type`. Type must be one of `Drug`, `Procedure`, `Condition`, `Measurement`, `Observation`. Validated up front. For this project: `tools/inference/all_terms_typed.csv` lists 107 chest-CT terms, all `Condition`.

### Outputs

Two complementary files written in one pass:

**File 1 — `--out` (required): JSONL, one window per line.** Model input.

```json
{
  "messages": [
    {"role": "system", "content": "<exact training SYSTEM_PROMPT>"},
    {"role": "user",   "content": "{\"entities\":[...],\"sentence\":\"<window>\"}"}
  ],
  "patient_id":   "<empi>",
  "note_id":      "<Report_Number>",
  "window_start": <int>,
  "window_end":   <int>
}
```

`system` content is byte-identical to what the fine-tune saw. `user` content is a JSON string matching the training format. Sidecar fields let you join model outputs back to source.

**File 2 — `--out-table` (optional): long-form TSV, one row per `(window, entity)`.** Analysis-ready stub.

| Column            | When filled        | Source                                                                |
|-------------------|--------------------|------------------------------------------------------------------------|
| `empi`            | pre-inference      | Patient identifier (string).                                            |
| `note_id`         | pre-inference      | Report identifier.                                                      |
| `window_start`    | pre-inference      | Char offset of the window in the source note.                           |
| `window_end`      | pre-inference      | "                                                                       |
| `context`         | pre-inference      | The window text itself.                                                 |
| `entity`          | pre-inference      | The entity term matched by the catalog.                                 |
| `entity_type`     | pre-inference      | One of Drug/Procedure/Condition/Measurement/Observation.                |
| `llm_input`       | pre-inference      | The full `user` content JSON (same for every entity sharing a window).  |
| `llm_output_raw`  | **post-inference** | The full assistant turn (raw string, for audit).                        |
| `assertion`       | post-inference     | Parsed axis value.                                                      |
| `certainty`       | post-inference     | "                                                                       |
| `temporality`     | post-inference     | "                                                                       |
| `subject`         | post-inference     | "                                                                       |
| `fifth_axis`      | pre-inference      | Axis name for this type (`action`/`status`/`interpretation`/`clinical_status`; empty for Observation). |
| `fifth_axis_val`  | post-inference     | Predicted value for the 5th axis.                                       |
| `status`          | post-inference     | `ok` / `parse_error` / `entity_missing` / `no_response` (added by stage 3). |

Format: tab-separated, header row, `csv.QUOTE_MINIMAL` quoting (handles tabs/newlines/quotes embedded in clinical text).

### Windowing strategy

For each `(term × note)` hit:
1. Find every term-hit and tag with the sentence it lands in.
2. Cluster hits whose sentence indices differ by ≤ `ADJACENCY_SENTENCES` (default 1 — strict adjacency; default windows would otherwise overlap 80%).
3. Each cluster yields ONE window: larger of (cluster sentence range ± `SENTENCE_RADIUS=2` sentences) OR (±50 words around the cluster's hit char-range). Hard cap at `MAX_WINDOW_CHARS=1500`, edges snap to whitespace.
4. Attach every input term whose lowercase form appears in the window.

Both knobs are configurable per-call: `windows_for_note(note, terms, adjacency=2, sent_radius=3)`.

## Stage 2: Run inference (`run_llamacpp_inference.py`)

The runner loops over `inference_inputs.jsonl` and invokes `llama-cli` once per window, capturing the model's assistant turn into a responses JSONL.

### Basic usage

```bash
python tools/inference/run_llamacpp_inference.py \
  --model  /path/to/medgemma-1.5-4b-it.Q8_0.gguf \
  --inputs inference_inputs.jsonl \
  --out    model_responses.jsonl
```

### What it does per record

For each input line:
1. Parses the JSONL — extracts `system`, `user`, and the four sidecar IDs.
2. Invokes `llama-cli` with:
   ```
   llama-cli -m <model> -sys <system_prompt> -p <user_content> \
             -st -no-cnv --no-display-prompt --log-disable \
             -n 512 -c 4096 --temp 0.0 -ngl 999
   ```
   `-sys` and `-p` together activate Gemma's Jinja chat template (auto-loaded from the GGUF). `-st` (`--single-turn`) exits after one assistant turn. `--temp 0.0` is deterministic.
3. Extracts the first balanced JSON array from stdout (tolerant to trailing whitespace / noise).
4. Appends to `--out` JSONL:
   ```json
   {"patient_id":"...","note_id":"...","window_start":0,"window_end":366,
    "response":"[{\"entity\":\"...\",...}, ...]"}
   ```

If the model emits something that can't be parsed as a JSON array, the raw stdout is stored anyway — stage 3 will flag it as `parse_error`.

### All flags

| Flag             | Default        | Purpose                                                                |
|------------------|----------------|------------------------------------------------------------------------|
| `--model`        | required       | Path to GGUF file.                                                      |
| `--inputs`       | required       | Inputs JSONL from stage 1.                                              |
| `--out`          | required       | Responses JSONL.                                                        |
| `--llama-cli`    | `llama-cli`    | Path to the binary if not on PATH.                                      |
| `--n-predict`    | 512            | Max tokens to generate per call.                                        |
| `--ctx-size`     | 4096           | Context window in tokens.                                               |
| `--temp`         | 0.0            | Sampling temperature.                                                   |
| `--grammar-file` | (none)         | Optional GBNF to constrain output to a JSON array.                      |
| `--n-gpu-layers` | 999            | Layers offloaded to GPU. Use 0 for CPU-only.                            |
| `--resume`       | off            | Skip windows already present in `--out` (idempotent re-runs).            |

### Performance note

`llama-cli` reloads the model on every invocation. On Apple Silicon with Q8_0 this is ~1–3 s of fixed overhead per record. That's fine for hundreds; slow for thousands.

For larger jobs, use `llama-server` to keep the model loaded:

```bash
# In one terminal:
llama-server -m medgemma-1.5-4b-it.Q8_0.gguf -c 4096 -ngl 999 --port 8080
```

Then a small Python loop using the `/v1/chat/completions` endpoint produces the same responses JSONL ~10–50× faster. A sketch is in the comment block at the bottom of `tools/inference/run_llamacpp_inference.py`.

## Stage 3: Merge responses (`merge_inference_outputs.py`)

Joins the per-window responses back into the per-entity TSV.

```bash
python tools/inference/merge_inference_outputs.py \
  --table     inference_inputs.tsv \
  --responses model_responses.jsonl \
  --out       predictions.tsv
```

### What it does

1. Indexes responses by `(patient_id, note_id, window_start, window_end)`.
2. Parses each response as a JSON array; builds a `entity_lower → entity_dict` lookup per window.
3. Streams the input TSV and, per row, looks up the entity's prediction:
   - `ok` — entity found in the parsed response; axis columns filled.
   - `entity_missing` — response parsed but didn't include this entity (model dropped it). `llm_output_raw` filled.
   - `parse_error` — response wasn't valid JSON. `llm_output_raw` filled; axis cols empty.
   - `no_response` — no response found for this window.
4. Writes the same schema as input + a new `status` column.

Prints a summary like:
```
Merged 6 rows into predictions.tsv: ok=4, parse_error=0, entity_missing=1, no_response=1
```

### Match logic

Entity-name matching is case-insensitive (model may capitalize differently than the catalog). Extra entities returned by the model that aren't in the input row set are silently ignored — they would have to introduce new rows, which the merge step intentionally doesn't do.

## Configuration constants

In `tools/inference/build_inference_input.py`:

| Constant              | Value | Meaning                                                                 |
|-----------------------|------:|-------------------------------------------------------------------------|
| `MAX_WINDOW_CHARS`    | 1500  | Hard cap on any output window.                                          |
| `WORD_RADIUS`         | 50    | ±50 whitespace tokens from the cluster's hit char-range.                |
| `SENTENCE_RADIUS`     | 2     | ±2 sentences past the cluster's hit-sentences.                          |
| `ADJACENCY_SENTENCES` | 1     | Hits with sentence-index diff ≤ this merge into one cluster.            |
| `VALID_TYPES`         | {Drug, Procedure, Condition, Measurement, Observation} | SCHEMA3 entity types. |

`SENTENCE_RADIUS` and `ADJACENCY_SENTENCES` are overridable per-call via `windows_for_note(..., adjacency=N, sent_radius=N)`.

## Optimizations specific to this stack

### Already in place
- **Q8_0 quantization.** Effectively lossless for structured-output tasks at this model size. No degradation concerns relative to Q4 variants.
- **Multi-entity-per-window records.** Each cluster of nearby terms shares one window and one model call. ~9× fewer calls vs one-entity-per-record on dense paragraphs.
- **Deterministic decoding** (`--temp 0.0`). Reproducible outputs; no run-to-run variance.
- **Verbatim system prompt.** Byte-identical to what the fine-tune saw. Do not paraphrase — the fine-tune anchored on the exact text.

### Worth doing for larger jobs

1. **Switch to `llama-server`.** The biggest win. Eliminates per-call model reload (~1–3 s saved per record on Apple Silicon Q8_0). See the comment block at the bottom of `run_llamacpp_inference.py`.

2. **GBNF-constrained output.** Pass `--grammar-file <path>` to force valid JSON-array structure. A starter GBNF for the SCHEMA3 axis vocabulary is a worthwhile add — it eliminates `parse_error` rows entirely at the cost of negligible generation overhead. (Not bundled here; happy to write one if you want.)

3. **Reduce `--ctx-size`.** Default 4096 is conservative. Your windows are capped at 1500 chars (~450 tokens); a `--ctx-size 1024` will be faster and use less memory. Just verify your longest window + system prompt fits.

4. **`--n-predict` ceiling.** 512 tokens is plenty for the largest entity list. If you ever see truncation (final `]` missing), raise it.

### Probably not worth it
- Lower quantization (Q4_K_M, AWQ). You're already at Q8_0; going lower trades ~2× speed for measurable degradation on structured output. Stay with Q8_0 unless RAM-constrained.
- Speculative decoding. Marginal benefit on a 4B model.

## Known concerns / behaviors

### Substring matching can over-match short terms
`_term_matches` uses unbounded case-insensitive substring search. Catalog entries that may over-fire:
- `mass` → matches "massive", "Massachusetts".
- `halo` → could match non-medical "halo" in social hx.
- `cyst` → matches "cystic" (which is also in your catalog — harmless duplication).
- `cavitation` → matches "cavitations".

The fine-tune saw similar noise, so this usually works. If precision matters more than recall on a given term, add `\b` word boundaries selectively in `_term_matches`.

### Surface-form duplicates are intentional
Catalog has `ground glass` / `groundglass` / `ground-glass`, `tree-in-bud` / `tree in bud`, etc. as separate entries. Substring matching is exact (case aside), so all variants are needed to cover real-world spelling. **Do not dedupe.**

### Typos in the source CSV
`pleuroparencyhmal` and `hypersnsitivity` are misspellings preserved from the original CSV — kept in case your notes contain the same typos. Add the correct spellings if your notes use them.

### Sentence splitter quirks
`\n+` is a terminator alongside `[.!?]\s+`. Bullet-point sections produce many short "sentences." The ±2 sentence rule covers a few of these, and the ±50-word rule rescues short ones. Works fine in practice but worth knowing for debugging.

### Training/inference distribution shift — measured, not just argued
The fine-tune was trained on per-hit windows with a 3-sentence rule
(`tools/training/extract_windows.py`); inference uses sentence-cluster anchoring with a
5-sentence rule. Profiling the 6431 windows in `inference_inputs.jsonl` against the training
envelope in `../../tools/training/STEAM_distribution_report.md` shows the dimension the model
actually reads — text length — matches:

| Metric          | Training median (p5–p95) | Inference median (p5–p95) |
|-----------------|--------------------------|---------------------------|
| Word count      | 100 (75–181)             | 100 (65–123)              |
| Char length     | 684 (514–1270)           | 711 (479–916)             |
| Sentence count  | 10 (3–15)                | 17 (11–24)                |
| Entities/window | 3 (p95 4)                | 3 (p95 8)                 |

- **Word/char length match almost exactly**, and the inference tail is *tighter* (no
  over-long windows). The ±50-word rule (identical to training) dominates window sizing, which
  is why length lands on-target.
- **The sentence-count gap (17 vs 10) is a splitter artifact, not a real difference.**
  Line-broken radiology reports fragment into more short "sentences" under the inference
  splitter's `\n+` rule, but the model never sees sentence boundaries and reads the same ~100
  words.
- **Entity density is the one genuine shift** — 32% of windows exceed the training p95 of 4
  entities (max 17 ≈ training max 16). On the 22 windows with sampled responses (spanning the
  5–8-entity bucket) the model dropped 0 of 101 entities with 0 parse errors, so it is tolerated
  in practice. To close this definitively, count `status=entity_missing` in `predictions.tsv`
  after a full run, bucketed by entities-per-window.

Because word/char length already match at the defaults, the defaults are the right setting.
Lowering to `sent_radius=1, adjacency=0` would *shorten* an already on-target center — don't
reach for it unless a full-run `entity_missing` check shows a problem.

### Training coverage gaps
From `../../EXAMPLES_BY_TEMPORALITY.md`:
- Drug · `unknown`, Procedure · `unknown`, Measurement · `unknown` — **0** training records.
- Observation · `unknown` — only 2 records.

Not directly relevant for chest-CT (all your terms are `Condition`, which has full temporality coverage). But if you later add Drug/Procedure/Measurement/Observation terms, expect weaker performance on `unknown` temporality.

## File index

| File                                                         | Purpose                                                 |
|--------------------------------------------------------------|---------------------------------------------------------|
| `tools/inference/build_inference_input.py`                             | Stage 1: build JSONL (model input) + optional long-form TSV. |
| `tools/inference/run_llamacpp_inference.py`                            | Stage 2: batch-run `llama-cli` over the JSONL.           |
| `tools/inference/merge_inference_outputs.py`                           | Stage 3: merge responses into the TSV; emit `predictions.tsv`. |
| `tools/inference/all_terms_typed.csv`                                        | 107 chest-CT terms, all typed `Condition`.              |
| `tools/inference/all_terms.csv`                              | Original input list (no type column).                    |
| `../../prompts/STEAM_demo_dataset.jsonl`                     | Training data the fine-tune was built on.                |
| `../../DATASET_CURATION.md`                                  | How the training data was constructed.                   |
| `../../SCHEMA3.md`                                           | Axis definitions and cue-word lists.                     |
| `../../EXAMPLES_BY_TEMPORALITY.md`                           | Hand-curated examples from the training data per type × temporality. |
| `tools/training/extract_windows.py`                                   | Training-pipeline window extractor (NOT used at inference; retained for provenance). |
