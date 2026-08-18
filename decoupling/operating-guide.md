# Decoupled serving — documented operation

Sourced from `docs/cli.md` and the ADRs in the LARQL tree, not inferred from
`--help`. Flags below are the ones that materially change behaviour.

## Shard the model across nodes

| Flag (server) | Effect |
| --- | --- |
| `--layers START-END` | Load and serve only these layers. Pages outside the range are never faulted in, so RSS scales with shard size. |
| `--experts START-END` | Serve only this expert-ID range (MoE filter). Mutually exclusive with `--units`. |
| `--units PATH` | Per-(layer, expert) ownership manifest for fine-grained splits. |
| `--moe-shards SPEC` | Server-side fan-out: this server dispatches expert calls onward. Combine with `--layers` for 2D layer × expert sharding. |

Layer sharding is the mechanism that lets a model exceed any single node's
memory: each node faults in only its own range.

## Make the server fast

| Flag | Effect |
| --- | --- |
| `--warmup-hnsw` (needs `--hnsw`) | Eagerly build the HNSW expert index for every owned layer at startup. Produces the `built N units (layers × experts/shard)` log line. |
| `--warmup-walk-ffn` | Pre-load inference weights and prefetch owned layer mmap pages at boot. |
| `--uds-path PATH` | Unix domain socket alongside TCP, for same-host shard clients. Removes TCP overhead when client and server share a machine. |
| `--release-mmap-after-request` | `madvise(DONTNEED)` after each request. Strict on Linux, advisory on Darwin. |
| `--max-gate-cache-layers N` | LRU cap on decoded f16 gate layers. |

## The wire protocol

`POST /v1/walk-ffn` has two modes:

- **Features-only (default).** Client posts one `[hidden]` residual; server runs
  gate KNN and returns feature indices + scores. The client still needs
  `up_features.bin` and `down_features.bin` locally.
- **Full-output** (`"full_output": true`). Client posts `[seq_len × hidden]`
  row-major plus `seq_len`; server runs the whole FFN path and returns
  hidden-size output per layer. This is what `larql run --ffn URL` uses — server
  holds all FFN weights, client holds only attention.

Privacy consequence: both modes put the residual stream on the wire. Full-output
minimises local weights and maximises traffic; features-only keeps more weights
local and sends less per call. Neither keeps the residual local.

A gRPC `WalkFfn` RPC mirrors the HTTP endpoint. QUIC and HTTP/3 transports exist
(ADR-0010, ADR-0019).

## Self-assembling grid

Shipped, not aspirational — ADR-0004 (grid), ADR-0011 (self-balancing, shipped
2026-05-13/15), ADR-0014 (hot-shard replication).

    larql-router --grid-port 50052            # router
    larql serve <vindex> --join grpc://router:50052 --public-url http://node:8080

`--join` requires `--public-url`; `--grid-key` (or `LARQL_GRID_KEY`) authenticates.

Router subcommands: `status` (servers, shards, gaps), `gaps` (coverage gaps per
model), `drain` (tell a server to drain and exit), `assign` (force a layer range).

The rebalancer handles drain-then-reassign, shard replication, hot-shard
load-rate elevation with two-threshold hysteresis, stale-heartbeat eviction, and
per-layer latency imbalance detection. Mode B means a node advertises capacity
("I have 24 GB, what do you need?") and the router provisions a range — before
it, ranges had to be hard-coded per node.
