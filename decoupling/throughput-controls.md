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

**Decoupling is the only sane way to run the 26B class on this machine.**
Against the realistic local alternative — `gemma4:26b` through Ollama —
decoupled serving is 1.7× faster *and* costs a fraction of the memory. The
Ollama run pushed swap to 8.3 GB and tripped the watchdog guarding it; the
decoupled run left swap flat at ~1.5 GB with the expert server resident in
2.9 GB. Same weight class, same machine, same prompt.

Against the naive LARQL local path it is 26× faster (1.6 vs 0.06 tok/s), and
that path also pays a 306-second prefill on a 38-token prompt.

The gain is amortisation, not distribution. The expert server builds HNSW units
once at boot (90 units in 4.6 s), pre-loads inference weights, and keeps gate
caches hot across requests; the monolithic path redoes that inside the decode
loop. **No second machine is required to collect this** — both processes ran on
one host.

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
