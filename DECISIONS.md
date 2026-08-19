# Decision history

### 2026-08-18 — Calibrate in memory, compile once

**Status:** Active
**Decision:** Search edit parameters by installing the edge on a loaded model,
then run `compile` once with the chosen values.
**Reason and evidence:** `compile` peaks near 26 GB and takes minutes per trial
on a 16 GB machine; the in-memory harness loads in ~2 s. Twenty-one parameter
trials cost less than one compile.
**Recorded by:** measured, 2026-08-18.

### 2026-08-18 — Every edit is verified against controls

**Status:** Active
**Decision:** No edit is accepted without two control prompts sharing the
trigger's grammatical shape, with BOS prepended.
**Reason and evidence:** Leakage appears first on same-shaped questions — at
layer 25, alpha 0.6 kept the fact at 99% while Paris fell to 28% and Tokyo to
9%. Separately, a missing BOS produced echo output that reads exactly like a
failed edit, and a name-mismatched checkpoint silently loaded random weights.
Two false results before the harness was trustworthy.
**Recorded by:** measured, 2026-08-18.

### 2026-08-18 — This is research, not an OPA feature

**Status:** Active
**Decision:** Weight editing stays in this repo and out of OPA.
**Reason and evidence:** Cost per fact is a calibration plus a multi-minute
compile; capacity collapses around ~1400 edits; nothing survives a model
upgrade; episodic recall is impossible. A system prompt covers a stable fact
core for ~200 cached tokens, and retrieval covers what changes.
**Revisit when:** a need appears that requires a fact present with zero prompt
overhead and no retrieval step.

### 2026-08-18 — Build from upstream HEAD, never a PR branch

**Status:** Active
**Decision:** Reproduce on `upstream/main` first. Only bisect to a branch after
a failure is confirmed on current code.
**Reason and evidence:** Work proceeded on `pr181`, fetched while chasing an
issue-tracker reference. It was **498 commits behind main**. On identical inputs
— same vindex, same servers, same command — that branch produced multilingual
noise and main produced the correct answer. Hours of diagnosis, and five
patches, were spent on behaviour that current code does not exhibit.
**Rollback path:** None needed; checking `git rev-list --count <branch>..main`
takes one second.

### 2026-08-18 — Extract via the documented safetensors path

**Status:** Active
**Decision:** `larql model pull <repo>` then `larql extract <snapshot> --quant q4k`.
Do not extract from Ollama GGUF blobs.
**Reason and evidence:** The GGUF converter has no MoE support — it silently
produced a dense-shaped vindex with `moe: None`, dropping 128 experts × 30
layers, and emitted no layer types (verified on main's own binary at PR #269),
so the architecture fell back to a hardcoded stride. The documented path
produced a correct 15 GB artifact with `layers/` and full MoE config on the
first attempt.
**Affected work:** The `layer_types` GGUF patch repairs a path we no longer use.

### 2026-08-18 — One expert server per machine

**Status:** Active
**Decision:** Serve all experts from a single process per host; shard across
hosts, not within one.
**Reason and evidence:** Two shard servers on one machine duplicate the mmap and
process overhead with no benefit — the demo's 0-63 / 64-127 split models two
machines. `--uds-path` removes TCP for same-host clients.

