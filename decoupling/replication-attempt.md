# Decoupled serving on this machine — what actually happened

Attempted 2026-08-18 on a 2020 M1 MacBook Pro, 16 GB, using the published
Gemma 4 26B-A4B slices. The architecture ran; the published artifacts did not
produce coherent output.

## The correct invocation

For a MoE model the client needs an **expert shard map**, not dense FFN
routing. `--ffn URL` is the dense path and fails on a MoE model with
`decode_token_with_moe returned None during prefill` — that error means the
client had no expert routing, not that the server is broken.

    larql serve <vindex> --experts 0-63   --port 8081
    larql serve <vindex> --experts 64-127 --port 8082
    larql run   <vindex> --moe-shards "0-63=http://localhost:8081,64-127=http://localhost:8082" \
                "The capital of France is"

Dispatch strategies: `--moe-dispatch streaming` (default, one stream per shard,
30 sequential round trips per decode token, exact) or `batch` (all layers in one
round trip, approximate; pair with `--moe-predispatch-iters 2`).

## Measured, 26B-A4B, both shards on localhost, attention on CPU

| Metric | Value |
| --- | --- |
| Prefill (TTFT) | 18,509 ms |
| Decode | 0.6 tok/s |
| Expert invocations | 3,600 (30 layers × top-8 × 15 tokens) |
| Bytes sent | ~9,900 KB |
| Bytes received | ~9,900 KB |

**~660 KB of residual stream per token, each way.** A 1,000-token turn moves
roughly 660 MB. That is the figure any network-boundary decision rests on.

The reference demo reports 24-25 tok/s on localhost with attention on **GPU**;
ours ran on CPU (`--metal` not used), which is the likely bulk of the gap.

## The published slices are not usable as-is

| Problem | Detail |
| --- | --- |
| Stale attention format | Client ships `attn_weights_q4k.bin`; the loader wants `interleaved_kquant.bin` / `interleaved_q4k.bin`. Copying the 360 MB interleaved file from the expert-server slice gets past it. |
| Missing declared files | Both slices' `index.json` list `gate_vectors.bin` and `down_meta.bin` in checksums. Neither repo contains them — confirmed against the HF tree API, not just the download. |
| Incomplete `larql pull` | `larql pull` fetches only browse-level files and silently omits inference weights. Fetching the repo directly over HTTP works. |

Output with these slices is multilingual noise, consistent with expert
selection running without gate data.

## Conclusion

The mechanism reproduces: attention local, router local, experts remote and
sharded, on one machine. Coherent output requires building the vindex from
model weights rather than consuming the published slices.
