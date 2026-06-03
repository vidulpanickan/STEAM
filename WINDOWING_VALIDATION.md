# Windowing validation — inference vs. STEAM training distribution

Does the inference windowing in `build_inference_input.py` produce text that matches the
distribution the MedGemma fine-tune was trained on? The fine-tune anchored on the exact text
shape of its training windows, so inference inputs must land in the same envelope or accuracy
degrades.

**Verdict: the strategy is sound, and empirically validated at its default parameters.** There
is one bounded watch-item (entity density per window) that the model currently absorbs without
dropping entities.

## How the two pipelines differ (by construction)

| Dimension | Training (`tools/training/extract_windows.py`) | Inference (`tools/inference/build_inference_input.py`) |
|---|---|---|
| Anchor | per-hit (one window per hit) | **per-cluster** (merges hits in adjacent sentences → ~9× fewer windows) |
| Sentence radius | hit **±1** sentence (3-sentence span) | cluster **±2** sentences (`SENTENCE_RADIUS=2`, 5-sentence span) |
| Word radius | ±50 words | ±50 words (**identical**) |
| Char cap | 1500 | 1500 (**identical**) |
| Sentence splitter | `(?<=[.!?])\s+(?=[A-Z])` + blank-line `\n\s*\n` (capital-letter lookahead; splits only on blank lines) | `(?<=[.!?])\s+` + **any** newline `\n+` (no lookahead; splits on every newline) |

Three things drift: anchoring (per-hit → cluster), sentence radius (±1 → ±2), and the splitter
regex. The question is whether those drifts move the *text the model actually reads* out of the
training envelope.

## Evidence

Profiling the 6431 windows in `inference_inputs.jsonl` with the same metric definitions used for
`../training/STEAM_distribution_report.md` (the report measures the `sentence` field of the 2910
training records using the inference splitter, so the word/char metrics are apples-to-apples):

| Metric | Training median (p5–p95, max) | Inference median (p5–p95, max) | Read |
|---|---|---|---|
| **Word count** | 100 (75–181, 263) | **100** (65–123, 228) | ✅ center identical; inference tail *tighter* (no over-long windows) |
| **Char length** | 684 (514–1270, 1497) | **711** (479–916, 1498) | ✅ within ~4% of training median; tail tighter |
| **Sentence count** | 10 (3–15, 29) | 17 (11–24, 86) | ⚠️ ~70% higher — splitter artifact, not a real text difference |
| **Entities/window** | 3 (p95=4, max=16) | 3 (p95=**8**, max=17) | ⚠️ same center, heavier tail: 32% of windows have >4 entities |
| **Entity types** | 68% Condition (mixed) | 100% Condition | ➖ acknowledged; Condition is the best-covered type, full temporality |

### Why the sentence-count divergence is harmless
The model never sees sentence boundaries — it reads raw window text. Sentence count matters only
as a window-sizing input. Radiology reports are heavily line-broken, and the inference splitter
breaks on every `\n`, so the *same ~100 words* get counted as 17 short "sentences" instead of 10.
Crucially, with `SENTENCE_RADIUS=2` over many short sentences the sentence-span rule produces a
small span, so the **±50-word rule (identical to training) dominates and sets the window** —
which is exactly why word/char length land on-target. The splitter divergence is self-correcting.

### Why the entity-density tail is tolerated
32% of inference windows carry >4 entities (training p95=4), from cluster-merging plus a dense
100%-Condition catalog. But (1) inference max (17) ≈ training max (16), so it stays
in-distribution, and (2) the model's actual entity-drop rate, measured on the 22 windows with
responses in `model_responses.top50.jsonl` (spanning the 5–8-entity bucket), was **0 of 101**
requested entities dropped with **0 parse errors**. Even above the training p95, the model
currently returns every entity. (Caveat: only 22 windows had sampled responses; a full-corpus
`status=entity_missing` count would make this definitive.)

## Bottom line

- The dimension that governs what the model reads — **word/char length — matches the training
  envelope almost exactly at the default parameters**, and is slightly *safer* (no over-long
  windows). This is the strongest signal that the strategy is sound.
- The two flagged drifts (sentence count, entity density) are respectively a counting artifact
  and a heavier-but-in-range tail the model currently absorbs with 0% drop.
- The defaults are the right setting. Lowering to `sent_radius=1, adjacency=0` to "approximate
  training" would *shorten* an already on-target center — don't reach for it unless a full-run
  `entity_missing` check shows a problem.

## Reproducing these numbers

- Training envelope: `python3 tools/training/analyze_steam_distribution.py` → regenerates
  `tools/training/STEAM_distribution_report.md`.
- Inference profile: the word/char/sentence/entity stats above come from profiling
  `tools/inference/inference_inputs.jsonl` (the JSONL emitted by stage 1) with the same metric
  definitions, plus the entity-drop check joins `model_responses.top50.jsonl` back to the input
  windows by `(patient_id, note_id, window_start, window_end)`.

## Open follow-ups

1. **Definitive density check** — after a full model run, count `status=entity_missing` in
   `predictions.tsv` bucketed by entities-per-window. If the drop rate stays ~0 in the 5–8 and
   9+ buckets, the watch-item is closed for good.
2. **Defensive guard (probably unnecessary)** — split any window with >~12 entities into two
   records in `windows_for_note`. The data says it is not needed today; only worth it if (1)
   reveals drops in the dense tail.
