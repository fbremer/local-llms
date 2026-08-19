# Ollama on this machine — operating facts

2020 M1 MacBook Pro, 8-core GPU, 16 GB unified memory, `iogpu.wired_limit_mb =
13312`. Ollama 0.32.14.

## `ollama ps` is not authoritative

It was wrong in **both directions** during one session:

- reported nothing loaded while three `llama-server` processes held ~40 GB
- reported a model loaded at 12 GB when no `llama-server` process existed

The truth is the process table:

    ps -Ao pid=,rss=,command= | grep '[l]lama-server'

Every stall, OOM, and bogus measurement in that session traced back to trusting
`ollama ps` for unload verification. Note the process is **`llama-server`** —
grepping `ollama runner` finds nothing and looks clean.

## Orphaned runners persist and are invisible

An `llama-server` holding 8.56 GB survived 3.5 hours across many unload calls.
Everything loaded afterwards competed with it, producing steadily worse
throughput and one apparent hang. `keep_alive: 0` does not reliably reap.

Before any measurement: `pkill -9 -f 'llama-server'`, then confirm the count is
zero by process, not by API.

## GPU offload caps at ~89%

`num_gpu` is the count of **transformer layers** placed on the GPU, not GPU
cores. Requesting all 52 layers yields 88–89% placement; full offload never
happens. Layers on GPU consume *wired* memory bounded by
`iogpu.wired_limit_mb`; what must fit is weights + that layer share of the KV
cache + the compute graph's scratch buffers.

Exceeding it gives:

    error: Insufficient Memory (kIOGPUCommandBufferCallbackErrorOutOfMemory)
    llama_decode: failed to decode, ret = -3

The model loads and then dies on first decode. Lowering `num_ctx` does not help
when the *weights* are what overrun — only lowering `num_gpu` does.

## `PARAMETER stop` is additive

A Modelfile with `FROM <existing-tag>` **inherits** the base's stop strings and
cannot remove them; adding more only lengthens the list. To change a stop list,
rebuild from the blob: dump `ollama show --modelfile`, drop every `PARAMETER`
line, keep the `FROM` lines and the `TEMPLATE` block **verbatim**, then
re-declare parameters.

Do not strip `#` lines when doing this — some templates contain a literal `#`
inside the quoted block, and removing it breaks the `"""` string with
`Error: unexpected EOF`.

## `FROM <blob path>` duplicates the weights

Building a Modelfile that references a blob by absolute path re-imports it under
a new digest instead of referencing the existing one. Two tags built this way
silently cost 24.7 GB of duplicate weights.

It does *not* always duplicate — when the re-imported content hashes identically
it dedupes. Check `du -sh ~/.ollama/models/blobs` before and after, and delete
the now-redundant source tag to reclaim the original.

## Runtime options beat rebuilding

`num_ctx` and `num_gpu` are per-request options and are also exposed in the GUI.
`stop` can be overridden per request too. The Modelfile value is only a default.
The **only** thing that genuinely requires a Modelfile is changing which blobs
load — e.g. dropping a vision projector.

## Dropping vision saves ~4 GB

Multimodal GGUFs ship as two blobs: text weights and an `mmproj` projector.
Omitting the second `FROM` line produces a text-only model — `Capabilities`
drops `vision`, and the ~3.9 GB projector reservation disappears. Text quality
is unaffected; the projector was never part of the text model.

The projector is expensive at runtime: `[mtmd] estimated worst-case memory usage
of mmproj is 3980.08 MiB`. On a 16 GB machine that turns a working text model
into a swapping one.

## Measuring throughput correctly

Every one of these produced a wrong number before being controlled:

| Confound | Effect |
| --- | --- |
| Battery + Low Power Mode | roughly halves decode throughput |
| Machine sleeping mid-run | looks like a collapse to 0.05 tok/s |
| Orphaned `llama-server` | progressive degradation across a session |
| `pkill` between samples | forces a cold 12 GB re-read; prefill swings 10–24 s |
| `num_predict` ~50 on a cold load | warmup dominates; reads ~50% low |

Method that works: AC power, kill orphans, load once, discard the first sample,
then take the median of three warm samples. Spread drops from ~0.8 to ~0.2
tok/s — below that, differences are not real.
