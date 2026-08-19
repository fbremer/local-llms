# Muse-Glimmer-30B on a 16 GB M1

A dense 30B multimodal model, running usably on this machine at ~3.5 tok/s.

## Provenance

Four hops. Only the last is ours.

1. **`meta-models/Muse-Glimmer-30B`** — official, Apache 2.0. 55.5 GB
   full-precision safetensors. Dense text model: 52 layers, hidden 6656,
   intermediate 19968, GQA 32Q/2KV, head_dim 128, vocab 202048. Plus a ~1.8B
   ViT-G/14 perception encoder.
2. **`unsloth/Muse-Glimmer-30B-GGUF`** — third-party quants by Unsloth AI
   ("Dynamic 2.0", imatrix calibration). We use `UD-IQ3_XXS` (12.23 GB) and
   `UD-Q2_K_XL` (11.59 GB), plus the 3.58 GB mmproj.
3. **`ollama pull hf.co/unsloth/...`** — Ollama's HF passthrough, which
   auto-generates a Modelfile from GGUF metadata. This is where the broken stop
   list originates.
4. **Local Modelfile** — rebuilt to fix the stop list, pin `num_gpu`, and (for
   the text-only tags) omit the projector.

No quantizing, compiling, or weight conversion was done locally.

**Caveat on the vendor's accuracy claim.** Unsloth reports "1.0% degradation
across 15 benchmarks" for their *K-Quant-17GB* variant. That is not what runs
here — `IQ3_XXS` at 12.23 GB and `Q2_K_XL` at 11.59 GB are substantially more
aggressive. No quality benchmark has been run locally; validation is a handful
of trivial prompts.

## Ollama's official library does not fit

The registry's smallest variant is 17 GB (`30b-nvfp4`); everything else is 18 GB
or more, and the sub-18 GB options are MLX-format. Nothing there clears the
13312 MB wired limit. Third-party low-bit quants are the only path on 16 GB.

## Two packaging defects, neither hardware

**Self-halting stop list.** The auto-generated Modelfile is harmony format
(`<|start|>`, `<|message|>`, `<|channel|>`) and lists `<|start|>` and
`<|message|>` — the model's own reply markers — as stop strings. Generation
halts after ~3 tokens with empty content, which reads as a hardware failure.
Present in **every** quant in the repo, so it is systematic. Fix: rebuild the
stop list as `<|eot|>` and `<|return|>` only.

**Metal OOM from over-eager placement.** Ollama pins ~94% of a 13 GB model to
GPU and leaves no room for compute buffers. Fix: `num_gpu 52` still lands at
~89% and works; the failure came from the default, not from asking for more.

Ollama selects `gguf_chat_template` with `renderer="" parser="" harmony=null` —
there is no architecture-aware layer for `muse_glimmer`, which is why a bad
template has nothing to fall back on.

## Measured throughput

Text-only tags, AC power, `num_gpu 52`, warm median of three samples:

| Context | `Q2_K_XL` (2-bit) | `IQ3_XXS` (3-bit) |
| --- | --- | --- |
| 8k | **3.76** | 3.52 |
| 32k | — | 3.38 |
| 64k | **3.53–3.72** | 3.12–3.28 |
| 128k | **2.75–2.80** | Metal OOM |

`Q3_K_XL` (12.44 GB) is unusable — 0.60 tok/s, 18.9 s prefill. It does not fit.

**The two viable quants differ by ~6–10% in speed, not 25%.** An earlier 25%
figure was an artifact of comparing them at mismatched `num_gpu`. The real
distinction is the context ceiling: Q2 reaches 128k, IQ3 stops at 64k.

At 128k, free memory drops to 3% with 4.5 GB swap — it runs, but with no
headroom for anything else on the machine.

## Working tags

    local/muse-glimmer:30b-iq3-text    3-bit, 64k default   (better precision)
    local/muse-glimmer:30b-q2-text     2-bit, 64k default   (reaches 128k)

Both carry `num_gpu 52`, the corrected stop list, and the model card's
recommended sampling (temp 1.0, top_p 0.95, top_k 64). They run bare — no
options required.

Vision-capable variants (`30b-q2`, `30b-iq3`, `30b-q3`) keep the projector.

Tags live under a `local/` namespace deliberately: these are hand-built
third-party quants and should not sit where registry models live.

## Vision costs more than it looks

One 896×896 image is **~1373 tokens** (a 1431-token request minus ~58 tokens of
text). Worse, the projector reserves ~3.9 GB on top of the 13 GB model — a
~17 GB working set on a 16 GB machine. Image turns swap; one observed request
entered `process_mtmd` and never returned.

Estimated image turn when it does complete: ~95 s prefill, ~160–225 s thinking,
~30 s answer — roughly 4½–6½ minutes.

If images are not needed, use the text-only tags. That is the single largest
memory win available.

## It is a reasoning model

The template pins "Reasoning strength: high", and the model card documents
`low | medium | high | xhigh` via system prompt. Setting `low` measured *worse*
(92 tokens vs 74) than leaving it unset, because the directive lives inside the
template's `{{ if .System }}` block — with no `SYSTEM` set, no directive is sent
at all. The working tags therefore declare no `SYSTEM`.

Practical consequence: ~2000 characters of thinking before any answer. A
three-sentence reply costs ~460 tokens and several minutes. Usable for
considered answers, not conversation.
