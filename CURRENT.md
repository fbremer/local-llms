# Current project state

## Outcome now

A verified, documented pipeline for surgical knowledge editing on Gemma 4 E2B,
and an indexed offline reference for the architecture's real shape.

## Current state

- Knowledge editing on E2B: works, envelope measured, results in `editing/`.
- Decoupled serving of 26B: works, artifact and numbers below.
- Canonical extraction is the documented safetensors path. The GGUF route is
  known-broken for MoE and layer typing.
- Build from upstream `main`. A PR branch produced multilingual noise on inputs
  where main produced correct answers.
- The four compile-path fixes are confirmed live on main and are being proposed
  upstream as two branches. See `tooling/larql-gemma4-fixes.md`.

## Active decisions

Fix E2B rather than replicate published results on variants this machine cannot
host. Calibrate in memory before paying for a compile — but only to choose the
layer; compile rescales alpha, so flag values do not transfer. Verify every
edit against two same-shaped control prompts.

## Non-negotiables

Editing requires full-precision weights; a quantized GGUF cannot be the base.
Never edit the final layer. Never trust an edit that has not been checked
against controls with BOS prepended.

## Decoupled serving

Works end to end on this machine. Gemma 4 26B-A4B answers correctly with
attention on Metal locally and all 128 experts served from a separate process:

    Attention: metal (GPU) local | Router: local | Experts: remote
    "Explain in three sentences why the sky is blue." → correct Rayleigh
    scattering answer, 59 tokens, 1.3 tok/s, ~330 KB per token each way.

Artifact: `~/.cache/larql/26b-a4b.vindex` (15 GB, q4k, 30 per-layer expert
files). Built with the canonical process — `larql model pull` then
`larql extract --quant q4k` from safetensors.

One expert server per machine. Two shard servers on one host duplicate the
mmap and process overhead for no gain; `--uds-path` removes TCP for same-host
clients.

The reference demo reports 24 tok/s on different hardware. Ours is not a
like-for-like comparison and no conclusion should be drawn from the gap.

## Decoupling earns its place on the ladder

Measured 2026-08-19, one prompt, this machine —
`decoupling/throughput-controls.md`:

| Model | Engine | Decode | Memory |
| --- | --- | --- | --- |
| `gemma4:e2b-8k` | Ollama | 19.1 tok/s | ~3 GB |
| `gemma4:e4b-8k` | Ollama | 9.4 tok/s | ~4 GB |
| 26B-A4B decoupled | LARQL | 1.6 tok/s | 2.9 GB server, swap flat |
| `gemma4:26b` | Ollama | 0.95 tok/s | swap to 8.3 GB |
| 26B-A4B fully local | LARQL | 0.06 tok/s | 306 s prefill |

Against the realistic local alternative for the same weight class
(`gemma4:26b` on Ollama), decoupled serving is 1.7× faster and costs a fraction
of the memory. Against LARQL's own local path it is 26× faster. The gain is
amortisation — HNSW units and inference weights built once at boot rather than
inside the decode loop — and it needs no second machine.

Cost: ~6× slower than `e4b-8k`. So `e4b` stays the default; 26B becomes
reachable, not interactive.

Rejected as explanations for our absolute throughput: `--moe-dispatch batch`
(doubles speed, destroys output; the documented `--moe-predispatch-iters 2`
remedy consumes the entire gain) and transport (`grpc://` with `--grpc-port`
and `--uds-path` matched HTTP byte for byte). The bottleneck is the sequential
per-layer dependency itself, ~21 ms per hop with neither end CPU-saturated.

## Extraction: use safetensors, not GGUF

The GGUF route silently drops MoE experts (produces a dense-shaped vindex with
`moe: None`) and emits no layer-type information, so the architecture falls back
to a hardcoded stride of 6 — correct for 26B by coincidence, wrong for E2B.
Verified against upstream main's own binary at PR #269.

## Open questions

- Will upstream take the compile-path fixes? Branches are authored and
  verified; not yet submitted.
- Whether a LAN split changes anything, given ~330 KB of residual per token
  each way.
- Why 21 ms per layer hop, when neither client nor server is CPU-saturated and
  the transport is not the constraint. This is the whole remaining gap to the
  reference throughput.

## Relationship to OPA

None established. Weight editing is a research capability here, not an OPA
feature: the prompt covers a stable fact core at trivial cost and retrieval
covers everything that changes. Recorded so the question is not reopened by
accident.

## Stop condition

A claim about model behaviour without a control run is not a result.
