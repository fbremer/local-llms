# What decoupling is for: the gemma4 ladder on this machine

2026-08-19, 2020 M1 MacBook Pro, 16 GB. One prompt throughout — "Explain in
three sentences why the sky is blue." Ollama runs use capped tags with
`think: false`, `temperature: 0`. LARQL runs use `--metal`, one expert server,
`--experts 0-127`, against `26b-a4b.vindex` (15 GB, q4k, built from
safetensors).

## The ladder

| Model | Engine | Decode | Memory behaviour |
| --- | --- | --- | --- |
| `gemma4:e2b-8k` | Ollama | **19.1 tok/s** | ~3 GB, swap flat |
| `gemma4:e4b-8k` | Ollama | **9.4 tok/s** | ~4 GB, swap flat |
| 26B-A4B **decoupled** | LARQL | **1.6 tok/s** | server 2.9 GB RSS, swap flat |
| `gemma4:26b` | Ollama | **0.95 tok/s** | **drove swap to 8.3 GB** |
| 26B-A4B fully local | LARQL | **0.06 tok/s** | 6.3 GB, 306 s prefill |

## The finding

**Results.** Against `gemma4:26b` under Ollama, on the same machine and prompt, decoupled serving was 1.7× faster. Ollama swap reached 8.3 GB. The decoupled run left swap near 1.5 GB, with the expert server at 2.9 GB resident. Against fully local LARQL the ratio was 26× (1.6 vs 0.06 tok/s), and that local path spent 306 s prefilling a 38-token prompt. The 1.7× and 26× figures are derived from those rates.

**Methods.** Both processes in the 1.6 tok/s run were on one host.

**Interpretation, not supported by a later result in this file.** Read off the rows above, decoupling appeared to be the only workable way to run this weight class here. The Q2 result below falls outside that claim.

**Post hoc interpretation.** The decode-rate gap is amortised setup: an HNSW build of 90 units in 4.6 s at boot, weights preloaded, and gate caches kept hot, against a monolithic path that repeats that work inside the decode loop. No control omitted the boot work. Restated as a hypothesis for a later run: if that boot work is skipped, the gap closes. That run has not been done.

## What it costs against the models we actually run

Decoupled 26B is ~6× slower than `e4b-8k` and ~12× slower than `e2b-8k`. So
this is not a replacement for the daily driver; it is a way to reach a heavier
class when the answer is worth ~40 seconds instead of ~7. The honest framing is
that `e4b` remains the default and 26B becomes reachable, not that 26B becomes
practical for interactive use.

## Reading the reference demo correctly

The source reports 22 tok/s local → 24 tok/s decoupled on its own hardware, and
the claim being made is *decoupling does not cost speed*. That claim replicates
here and lands harder: our ratio is 26×, not 1.09×, in decoupling's favour.

Our 1.6 tok/s must never be compared against the demo's 24 tok/s as though it
were the same experiment. That gap is hardware.

## Two hypotheses tested and rejected

**Dispatch strategy.** `--moe-dispatch batch` fires all layers in parallel and
doubles decode to 3.2 tok/s, but output collapses into prompt-echo. The
documented remedy `--moe-predispatch-iters 2` restores coherence and consumes
the entire speedup — back to 1.6 tok/s, with an answer that still gets the
physics wrong (claims the atmosphere absorbs light). Batch is not usable here.

**Transport.** `streaming` is documented as gRPC and had been pointed at an HTTP
port with no gRPC listener. Serving with `--grpc-port 9081 --uds-path` and
dispatching over `grpc://` gave 1.4 tok/s and byte-identical output. TCP
overhead is not on the critical path.

## Where the time goes

625 ms per token over 30 layers is ~21 ms per layer hop. During decode the
client sits at ~20% CPU and the expert server peaks at ~65% of **one** core of
eight. Neither end is saturated and the wire is not the constraint: the cost is
the sequential per-layer dependency — layer N must complete before N+1 can
dispatch. That is the bottleneck the source video names, and no flag in the
current CLI removes it.

## Correction to an earlier operating assumption

An expert server serving the full 15 GB vindex holds **2.4–2.9 GB resident**.
The vindex is mmapped and only sparsely faulted in. Earlier work treated
running expert servers as holding ~15 GB and killed them to reclaim memory
during a compile; that reclaimed far less than assumed. Mapped is not resident —
use `ps` RSS.

## Reproduce

    larql serve ~/.cache/larql/26b-a4b.vindex --port 8081 --experts 0-127
    larql run   ~/.cache/larql/26b-a4b.vindex \
      --moe-shards "0-127=http://localhost:8081" --metal -n 64 -v \
      "Explain in three sentences why the sky is blue."

Local control: same `larql run` with `--moe-shards` omitted. Binaries are the
fork's `target/release` build — `larql-server` is absent from `~/.local/bin`,
where the dispatcher fails with "Failed to exec larql-server".

Guard any `gemma4:26b` Ollama run with a swap watchdog; it reaches 8 GB of swap
on this machine. Stop the expert server when finished — it is a live listener
on 0.0.0.0.

## Decoupling pays for sparse, not for dense

Measured on this machine, same binary, same prompt:

| Model | Type | Local | Decoupled | Effect |
| --- | --- | --- | --- | --- |
| gemma4 26B-A4B | MoE | 0.06 tok/s | 1.6 tok/s | **26x faster** |
| gemma-3-4b | dense | 6.8 tok/s | 2.0 tok/s | **3.4x slower** |

MoE fetches top-8 of 128 experts per layer, so the 15 GB vindex never goes hot
and the expert server holds 2.9 GB resident. Dense pushes the whole residual
through the full FFN at every layer: nothing is sparse, nothing goes cold.

## Can the FFN half stay out of memory?

This is the question decoupling exists to answer — the FFN needs no GPU, so in
principle it need not occupy client RAM. On one machine, for dense, it does
anyway.

Dense FFN server RSS against a 5.2 GB vindex:

| Configuration | Peak RSS |
| --- | --- |
| default | 4.48 GB |
| `--release-mmap-after-request --max-gate-cache-layers 4` | 4.29 GB |

A 4% reduction, and `madvise(MADV_DONTNEED)` is **advisory on Darwin** by its
own help text. But this measurement does not show what it appears to: the
vindex is 5.2 GB on a machine with ~14 GB free, so nothing ever *forced* an
eviction. RSS reaching full weight size proves only that the OS was allowed to
cache it, not that it had to. The residency question is not settled by this
test.

**What settles it is bytes read per token, not residency.** Measured cold
sequential read on this SSD: **3.3 GB/s** (16.3 GB/s warm, from page cache).
Dense means every FFN weight is read every token, so a non-resident dense model
is bounded by weight-bytes ÷ read-bandwidth:

| Model | FFN at Q4 | Floor if non-resident |
| --- | --- | --- |
| Muse-Glimmer-30B | ~11.8 GB | 3.6 s/token = **0.28 tok/s** |
| 26B-A4B (MoE, top-8 of 128) | ~0.7 GB/token | 0.21 s = **4.7 tok/s** |

The MoE ceiling is well above our measured 1.6 tok/s, so I/O is not binding
there — the 21 ms per-layer round trip is. For dense it binds hard.

Read-only clean pages *are* cheaper than swap: they evict without writeback and
re-fault straight from SSD. That advantage is real and was missing from the
first version of this note. It does not rescue dense decoupling, because
decoupling does not change the byte count — the same 11.8 GB crosses the same
SSD per token whether llama.cpp reads it in-process or a LARQL server reads it
from another process on the same disk.

`--layers START-END` is the mechanism that does bound memory, but it works by
never faulting out-of-range pages, not by evicting. Sharding by layer on a
single host yields N servers whose RSS sums to the same total.

Since llama.cpp also mmaps its GGUF, dense decoupling on one box reduces to the
same paging behaviour as running the quant directly, plus round trips.

**The conclusion is not "dense decoupling is useless" — it is that dense
decoupling needs a second machine.** With one, the weights live in this RAM
either way. With two, the FFN occupies zero bytes here, which is the actual
claim under test. Residual traffic is ~330 KB per token each way; trivial on a
LAN, fatal over the internet (the reference demo drops to 1.8 tok/s on Fly.io).

## Applied: Muse-Glimmer-30B

Dense (52 layers, hidden 6656, intermediate 19968, no expert keys), multimodal,
55.5 GB in full precision, no published vindex. LARQL has first-class
`muse_glimmer` architecture support with judged semantics, so a vindex would
run correctly.

On this machine alone, decoupling does not unlock it — see above. The direct
path is the published GGUF (Q3_K_XL 12.4 GB, Q4_K_XL 14.8 GB) through
llama.cpp/Ollama, for reference against `gemma4:26b` at 17 GB → 0.95 tok/s and
8.3 GB of swap. With a second machine on the LAN, decoupled serving becomes a
real option.

## Provenance

All numbers from a local LARQL checkout at `caa6e266` (`gemma4-compile-fixes-v2`).
Those patches are compile-path only and should not affect serving, but this has
not been controlled: a clean upstream build and one re-measure would make these
figures portable.

`larql pull chrishayuk/gemma-3-4b-it-vindex` fetched 0 bytes of weights — stubs
only, reproducing the known browse-level-fetch defect. Use
`huggingface_hub.snapshot_download` instead.

## Correction: a dense 30B does run here, and beats decoupled 26B

Measured after the above. Muse-Glimmer-30B (dense), `UD-Q2_K_XL`, `num_gpu 42`,
2k context: **3.16 tok/s**, 468 tokens, `done_reason=stop`, correct answer,
peak swap 7.5 GB.

| Model | Engine | Decode |
| --- | --- | --- |
| `gemma4:e2b-8k` | Ollama | 19.1 tok/s |
| `gemma4:e4b-8k` | Ollama | 9.4 tok/s |
| **Muse-Glimmer-30B dense, Q2** | Ollama | **3.16 tok/s** |
| 26B-A4B decoupled | LARQL | 1.6 tok/s |
| `gemma4:26b` | Ollama | 0.95 tok/s |
| 26B-A4B local | LARQL | 0.06 tok/s |

**Result.** Muse-Glimmer-30B at `UD-Q2_K_XL` decoded at 3.16 tok/s, about twice the decoupled 26B in the table above.

**Derived value, under an assumption.** 0.28 tok/s divides the dense weight bytes by a cold sequential read rate, assuming the model is not resident and is reread from disk on every token. This Q2 run was largely GPU-resident, so it lies outside that assumption. The division is unchanged.

**Interpretation.** In these runs the dense-versus-sparse contrast showed up when the dense model did not fit, and the quantisation change is what changed the fit. That is an account of these runs, not a general cause.

Q3_K_XL (14 GB) does *not* work: it exceeds the 13312 MB wired limit, spills
65% to CPU and manages 0.06 tok/s. The usable window on this machine is a quant
that clears the wired limit with headroom for compute buffers.

### Two failures that were packaging, not hardware

**Broken stop list.** The GGUF's auto-generated template is harmony format
(`<|start|>`, `<|message|>`, `<|channel|>`) and lists `<|start|>` and
`<|message|>` as stop strings — the model's own reply markers. Generation halts
after ~3 tokens with empty content, which reads as a hardware failure and is
not. Override with `"stop": ["<|eot|>", "<|return|>"]`.

**Metal OOM from over-eager GPU placement.** Ollama pins 94% of a 13 GB model
to GPU against a 13312 MB wired limit, leaving nothing for compute buffers:
`kIOGPUCommandBufferCallbackErrorOutOfMemory`, `llama_decode: ret = -3`.
Lowering context does not help — the weights, not the KV cache, are the
problem. `num_gpu 42` (of 52 layers) fixes it at 71% GPU.

### Caveat: throughput is not latency

Muse-Glimmer is a reasoning model pinned to "Reasoning strength: high" by its
template. A three-sentence answer cost 468 tokens (2044 characters of thinking)
and **2:29 wall clock**. Usable for considered answers, not for conversation.

### The working tag

`muse-glimmer:30b` carries both fixes, so a bare call works with no options:

    FROM <UD-Q2_K_XL blob>
    FROM <mmproj blob>
    TEMPLATE """...harmony template, verbatim from the base..."""
    PARAMETER num_ctx 2048
    PARAMETER num_gpu 42
    PARAMETER stop <|eot|>
    PARAMETER stop <|return|>

The stop list had to be rebuilt from the blob rather than layered on top:
`PARAMETER stop` is **additive**, so a Modelfile with `FROM <existing-tag>`
cannot remove the base's `<|start|>` and `<|message|>` entries. Dump
`ollama show --modelfile`, drop every `PARAMETER`, keep the two `FROM` lines and
the `TEMPLATE` block verbatim, then re-declare parameters.

Verified bare: 2.56 tok/s, 423 tokens, `done_reason=stop`, 29%/71% CPU/GPU.
