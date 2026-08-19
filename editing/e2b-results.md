# E2B knowledge editing — measured results

Everything here was run on a 2020 M1 MacBook Pro, 16 GB, against
`google/gemma-4-e2b-it`. Measurements, not inference.

## The headline

A fact can be written into gemma4-e2b's weights and read back at 99.7%
confidence with unrelated knowledge untouched. Layer 25, alpha 0.15.

    baseline   "The capital of Atlantis is" → 'unknown' 46.3%
    edited     "The capital of Atlantis is" → 'Poseidon' 99.7%
    control    "The capital of France is"   → 'Paris'   100%
    control    "The capital of Japan is"    → 'Tokyo'   100%

## Operating envelope (layer × alpha sweep, gate_scale 1.0, slot 5000)

| Layer | 0.15 | 0.3 | 0.6 | Controls |
| --- | --- | --- | --- | --- |
| 15 | 0.0% | 0.0% | 0.0% | intact — nothing installs |
| 20 | 0.0% | 0.0% | 0.0% | **corrupted** ('said' 81%) |
| 25 | **99.7%** | 99.4% | 99.2% | clean to 0.3; **leaks at 0.6** (Paris 28%, Tokyo 9%) |
| 28 | 8.1% | 90.0% | 99.8% | intact |
| 30 | 1.6% | 52.4% | 99.3% | intact |
| 32 | 12.5% | 95.4% | 99.7% | intact |
| 34 | 1.7% | 0.1% | 0.0% | **destroyed** (Paris 3%→0%, Tokyo 3%→0%) |

Knowledge band is L25–L32. L25 is the efficient point — highest effect at the
lowest force. L34 (final layer) destroys unrelated facts and installs nothing.
Below L25 nothing installs; L20 damages output without installing.

The tool's defaults (`--layer 30 --alpha 0.3`) land at 52.4% — a weak spot for
this model. That, not a broken mechanism, is why the first compiled edit
appeared to fail.

## Method that made the sweep cheap

`larql compile` peaks near 26 GB and takes minutes per trial. Installing the
same edge directly on a loaded model — gate/up rows = trigger direction,
down column = answer direction, scaled by alpha — takes ~2 s to load and
milliseconds per trial. Twenty-one parameter combinations cost less than one
compile. Calibrate in memory, then compile once with the chosen values.

## Three traps that produce false negatives

1. **Missing BOS.** The shipped tokenizer config sets `add_bos_token: false`,
   but the model requires `<bos>` (id 2). Without it the model echoes prompt
   words — "The capital of France is" → 'France' 99.7% — which reads exactly
   like a broken edit. Every result before this was fixed was void.
2. **Silent random init.** Loading a checkpoint whose tensor names don't match
   the config leaves transformers randomly initializing and reporting
   `MISSING` weights. Output is noise at ~0.0% probability. Check for that
   warning before believing any result.
3. **No control run.** An edit's effect is only visible against the unedited
   model through the identical harness. The controls also catch leakage, which
   is the failure mode that matters — at L25/alpha 0.6 the fact installs at
   99% while Paris collapses to 28%.

## Scale limit

Sequential edits cause gradual then catastrophic forgetting; MEMIT-style
methods collapse around ~1400 edits (arXiv:2401.07453, arXiv:2403.07175).
This is a mechanism for a small, stable fact core — not a memory system.

## Compile alpha is not in-memory alpha

`larql compile` rescales the alpha you pass before writing (it reports the
effective value). The same nominal number is a much weaker edge through the
compile path than through an in-memory install:

    --alpha 0.15 --gate-scale 1.0   → effective 0.084 → 'unknown' 41.4%  (no edit)
    --alpha 1.0  --gate-scale 2.0   → effective 0.558 → 'Pose'    99.0%  (installed)

So the in-memory sweep above locates the *layer*, not the flag value. Pick the
layer in memory; find the alpha by compiling at the high end and checking the
controls.

The tool's defaults are stated in its own help text as tuned on Gemma 3 4B
(`g=1.0, α=0.3`, "the paraphrase-sweep sweet spot"). They under-install on E2B.
`--max-iters` defaults to 0, which disables the balancer that would otherwise
tune alpha into `[--floor, --ceiling]`; the help explains that too, as the Rust
forward pass being peakier than HF's.

## Status of the tooling fixes

Re-verified 2026-08-18 against upstream `chrishayuk/larql` main at PR #269.
All four compile-path defects are live on current main — this is not an
artifact of the stale branch the fixes were first written against.

    stock main     panic: assertion failed: index < dim   (edge.rs:131)
    patched main   installed at L25 slot 9000 → saved checkpoint
                   loads with 0 missing-weight warnings
                   Atlantis → 'Pose' 99.0% | Paris 98.1% | Tokyo 98.8%

See `tooling/larql-gemma4-fixes.md`.
