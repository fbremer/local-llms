# Decoupled attention — what the author demonstrated

Source: Chris Hay, "I Decoupled Attention from Weights - Gemma 4 26B",
2026-05-05 (youtube.com/watch?v=1jGR4zqpyKA). Numbers are his, on his machine.
Implementation lives in the LARQL repo itself — `larql slice`, `larql serve`,
`larql run --ffn URL`, and the `larql-router` / `larql-server` crates.

## The split

Attention stays local and is the only part needing GPU (~2 GB). The FFN — the
knowledge, and the bulk of the weights — becomes an HTTP service. Each layer,
each token, the client sends the residual stream out and reads the FFN's
contribution back. A 26B vindex is 16 GB; the client slice is ~2 GB after
carving off 14 GB of expert weights.

Sharding is by expert range (`0-63` on one port, `64-127` on another), by
layer, or both. He notes ~15,000 expert shards would be possible.

## Measured behaviour

| Configuration | Throughput |
| --- | --- |
| Gemma 3 4B, fully local, GPU | 83 tok/s |
| Gemma 4 26B, fully local, Metal | 22 tok/s |
| Gemma 4 26B, decoupled, both halves on localhost | 24-25 tok/s |
| Gemma 4 26B, decoupled to Fly.io (~120 miles) | **1.8 tok/s** |

Localhost decoupling costs nothing. Internet decoupling costs an order of
magnitude, because inference is sequential: 30+ layers, a round trip each, per
token. His conclusion is that LAN-local CPU boxes are the practical form.

## What crosses the wire

**~41 MB sent and ~41 MB received for one query.** In his words: "this is
literally... the residual streams going across the wire."

This is the number that decides whether decoupling is compatible with a
data-stays-local policy. Residual streams are not text, but they decode to it —
the same project's `DESCRIBE` and logit-lens machinery exists to read meaning
out of exactly these vectors, and knowledge editing works by matching against
them. Treat a residual stream leaving the machine as the text leaving the
machine.

## Relevance to a 16 GB machine

The demo's value is running a model too large to host — 26B against a ~2 GB
local slice. That requires a second machine holding the expert weights. On one
machine, decoupling only measures protocol overhead.

The published work is on 26B (MoE) and 31B (dense). Nothing has been
demonstrated on E2B, which is dense and small enough to run locally anyway.
