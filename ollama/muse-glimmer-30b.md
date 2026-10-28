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
aggressive. A blind comparison that includes these quants is in
[evaluation/summarisation-quality.md](../evaluation/summarisation-quality.md).
It did not resolve an ordering among the Muse quants.

## Ollama's official library does not fit

The registry's smallest variant is 17 GB (`30b-nvfp4`); everything else is 18 GB
or more, and the sub-18 GB options are MLX-format. Nothing there clears the
13312 MB wired limit. Third-party low-bit quants are the only path on 16 GB.

## Two packaging defects, neither hardware

**Self-halting stop list — an Ollama-only failure.** The GGUF's tokenizer
metadata is non-standard; llama.cpp loads it and warns:

    W load: setting token '<|message|>' (200023) attribute to USER_DEFINED
    W load: special_eot_id is not in special_eog_ids - the tokenizer config
            may be incorrect

llama.cpp warns and **generates correctly anyway**. Ollama's `hf.co/`
auto-conversion instead turns those markers into stop strings, so generation
halts after ~3 tokens with empty content — which reads as a hardware failure and
is not. Present in every quant in the repo.

Fix under Ollama: set the stop list to Meta's documented pair only —
`<|end_of_text|>` (200001) and `<|eot|>` (200008). **Never `<|eom|>`**: it ends a
message, not a turn, and stopping there collapses parallel tool calling.
Under llama.cpp: no fix needed — see the control below.

Mechanism, confirmed: Ollama does not read a Modelfile from the HF repo. HF
generates one with `@huggingface/ollama-utils`, which converts the Jinja
template to a Go template and **derives `PARAMETER stop` from special tokens it
finds**. Its `CUSTOM_TEMPLATE_MAPPING` has hand-written entries for Gemma2,
Llama 3, Mistral Small and others, but **none for harmony-style**
`<|start|>`/`<|message|>`/`<|channel|>` templates. With no mapping, the generic
converter emits the role markers as stop strings.

This is reported and unresolved upstream: `unsloth/Muse-Glimmer-30B-GGUF`
discussion #16 reports the identical failure with the same two quants on
**AMD ROCm/Ubuntu** — different hardware entirely, so it was never a Metal, RAM,
or wired-limit problem. Unsloth staff say elsewhere their GGUFs "do not 100%
work in Ollama ... due to potential chat template incompatibility issues".

**Metal OOM from over-eager placement.** Ollama pins ~94% of a 13 GB model to
GPU and leaves no room for compute buffers. Fix: `num_gpu 52` still lands at
~89% and works; the failure came from the default, not from asking for more.

Ollama selects `gguf_chat_template` with `renderer="" parser="" harmony=null` —
there is no architecture-aware layer for `muse_glimmer`, which is why a bad
template has nothing to fall back on.

## Measured throughput

Two regimes. **The second supersedes the first** — everything below the divider
was measured with an f16 KV cache and no flash attention, which is not the
recommended configuration.

### With flash attention + q8_0 KV (current)

`OLLAMA_FLASH_ATTENTION=1`, `OLLAMA_KV_CACHE_TYPE=q8_0`, AC power, `num_gpu 52`,
warm median of three:

| Context | `IQ3_XXS` |
| --- | --- |
| 64k | **3.66–3.69** |
| 128k | **3.29–3.32** |

**128k works.** Under f16 KV it Metal-OOM'd; halving the KV cache is what
changed that. `q4_0` KV gave 3.73–3.78 at 64k — ~2%, within noise, more headroom
and more quality risk. `LLAMA_ARG_FIT_TARGET=256` was a **null result**:
placement stayed at 89%.

### Superseded: f16 KV, no flash attention

| Context | `Q2_K_XL` | `IQ3_XXS` |
| --- | --- | --- |
| 8k | 3.76 | 3.52 |
| 32k | — | 3.38 |
| 64k | 3.53–3.72 | 3.12–3.28 |
| 128k | 2.75–2.80 | Metal OOM |

`Q3_K_XL` (12.44 GB) measured 0.60 tok/s with 18.9 s prefill **in this regime
only** — it has not been retested with q8_0 KV, and the flags that rescued 128k
may rescue it too. Do not treat it as settled.

**The two viable quants differ by ~6–10% in speed, not 25%.** An earlier 25%
figure was an artifact of comparing them at mismatched `num_gpu`.

Any ceiling here is a property of the configuration it was measured under, not
of the machine. Four things move it: KV cache type, flash attention,
`iogpu.wired_limit_mb` (13312 now, a tunable, not hardware), and context.

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

## Control: the documented llama.cpp path

Unsloth documents **llama.cpp only** — no Ollama instructions, no Modelfile. The
documented path works as shipped, with no stop-list surgery:

    llama-server --model <blob> --port 8001 --host 127.0.0.1 --no-webui --offline \
      -c 8192 -np 1 --load-mode none --cache-type-k q8_0 --cache-type-v q8_0 \
      --flash-attn on -b 512 -ub 512 -ngl 52 --context-shift --keep 4 \
      --temp 1.0 --top-p 0.95 --top-k 64

Measured on IQ3_XXS: **3.49–3.55 tok/s**, 11.3 GB resident, zero OOM,
`reasoning_content` parsed natively into its own field. Ollama on the same blob
and equivalent flags: 3.66–3.69 tok/s. **Performance-equivalent** — the choice
between them is tooling, not speed.

Two traps when running llama-server directly:

- `OLLAMA_KV_CACHE_TYPE` and `OLLAMA_FLASH_ATTENTION` are **Ollama** env vars and
  mean nothing to `llama-server`. Ollama translates them to `--cache-type-k/v`
  and `--flash-attn`. Omitting the CLI flags reintroduces the OOM.
- Defaults are wrong for a 16 GB machine: `-np 4` allocates four KV caches
  (~140k of context), and full offload without `--load-mode none` OOMs. Ollama's
  `LLAMA_ARG_FIT` does this fitting automatically; raw llama-server does not.

To recover Ollama's exact invocation for any model, run the server with
`OLLAMA_DEBUG=1` and grep the log for `llama-server`.

**Why Ollama is still the default here:** GUI integration, model management, and
automatic memory fitting. The Modelfile rebuild is a workaround for a layer the
vendor does not target — not a necessity of the model.

## Differential test: the template was never broken

Claim under test: whether Ollama's Jinja→Go template conversion produces a
different prompt than llama.cpp's native Jinja path — which would degrade
quality invisibly rather than crash.

Method: same quant (IQ3_XXS), same prompt, `temperature 0`, `top_k 1`,
`seed 42`, run sequentially (two 12 GB models do not co-reside on 16 GB).

| Path | Tokens | Finish | Content | Reasoning |
| --- | --- | --- | --- | --- |
| Ollama tag | 115 | stop | `Paris` | 473 bytes |
| llama.cpp `--jinja` | 115 | stop | `Paris` | 473 bytes |

**Byte-identical reasoning traces — same MD5 (`f2ad71c7…`).** At greedy decoding
with matched seeds, identical output means identical input. **The template
conversion is faithful; only the stop list was ever wrong.**

Consequence: an "Ollama adapter" for this model family is two lines, not a
template rewrite. Everything else in the local tags — vision strip, `num_gpu`,
`num_ctx` — is machine-specific tuning, not adapter material.

**Limits of this result.** One prompt, one quant, single-turn, no tool calls.
Multi-turn and tool-calling exercise template paths this does not touch. And
both paths may be faithfully reproducing a **stale** template: the embedded one
is 7,167 chars against the 9,992 Meta documents, and discussion #18 asking
Unsloth to sync it is open and unanswered. That defect is upstream of Ollama and
is not addressed by the stop-list fix.
