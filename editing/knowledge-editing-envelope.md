# Knowledge editing on Gemma 4 E2B — measured envelope

A knowledge edit installs one FFN feature: the gate and up rows at a chosen
slot are set to the trigger direction, and the down column at that slot is set
to the answer direction, scaled by `alpha`. The model then fires that feature
when the residual matches the trigger.

All numbers below come from installing the edge directly on loaded weights and
measuring next-token probabilities. Trigger: `The capital of Atlantis is`.
Answer: `Poseidon`. Controls: `The capital of France is` (Paris) and
`The capital of Japan is` (Tokyo). Baseline for the trigger is `unknown` at
46.3%.

## Layer sweep, gate_scale 1.0, slot 5000

| Layer | alpha 0.15 | alpha 0.3 | alpha 0.6 | Controls |
| --- | --- | --- | --- | --- |
| 15 | 0.0% | 0.0% | 0.0% | intact |
| 20 | 0.0% | 0.0% | 0.0% | **output corrupted** ('said' 81%) |
| 25 | **99.7%** | 99.4% | 99.2% | intact to 0.3; **leaks at 0.6** (Paris 28%, Tokyo 9%) |
| 28 | 8.1% | 90.0% | 99.8% | intact |
| 30 | 1.6% | 52.4% | 99.3% | intact |
| 32 | 12.5% | 95.4% | 99.7% | intact |
| 34 | 1.7% | 0.1% | 0.0% | **destroyed** (Paris 3% → 0%, Tokyo 3% → 0%) |

## What the sweep says

**The knowledge band is layers 25-32.** Below layer 25 the edge does not
install: the fact never appears at any alpha. Layer 20 is actively harmful —
it fails to install the fact *and* corrupts the output distribution.

**Layer 25 is the efficient point.** It reaches 99.7% at alpha 0.15, the lowest
force of any working configuration, with both controls at 100%. Because it
needs so little force, it also has the least headroom: at alpha 0.6 the edge
starts firing on unrelated capital questions.

**Layers 28-32 are the tolerant range.** They need alpha 0.3-0.6 but hold the
controls at 99% throughout, so they suit facts that need pushing harder.

**Never edit the final layer.** At layer 34 the edge does not install and
unrelated facts collapse — Paris and Tokyo fall from 99% to 0%. Writing into
the last layer corrupts the output projection for everything.

## Recommended starting point

    --layer 25 --alpha 0.15 --gate-scale 1.0

Verify every edit against at least two control prompts that share the trigger's
grammatical shape. Leakage shows up on same-shaped questions first — a capital
edit leaks into other capital questions long before it touches anything else.

## Scale limit

Weight editing does not scale. Sequential edits cause gradual then catastrophic
forgetting, with MEMIT-style methods collapsing around ~1400 edits
(arXiv:2401.07453; see also arXiv:2403.07175 on ROME collapse). Treat this as a
mechanism for a small, stable fact core — not a substitute for retrieval.
