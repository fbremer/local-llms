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

### 2026-08-18 — This is research, not an application feature

**Status:** Active
**Decision:** Weight editing stays in this repo and out of any application.
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


### 2026-08-19 — Muse-Glimmer-30B runs here; IQ3_XXS is the default

**Status:** Active
**Decision:** Run Muse-Glimmer-30B from Unsloth's `UD-IQ3_XXS` (3-bit) as
`local/muse-glimmer:30b-iq3-text`, text-only, `num_gpu 52`, 64k context. Keep
`UD-Q2_K_XL` as the long-context option because it alone reaches 128k.
**Replaces:** The initial pick of `UD-Q2_K_XL`, chosen on a size threshold
derived from a `Q3_K_XL` test run at the wrong GPU placement.
**Reason and evidence:** Warm medians of three, AC power, `num_gpu 52` —
Q2 3.76 / IQ3 3.52 at 8k; Q2 3.53-3.72 / IQ3 3.12-3.28 at 64k; Q2 2.75-2.80 at
128k where IQ3 Metal-OOMs. The two differ by ~6-10%, not the 25% first reported;
that figure was an artifact of comparing them at mismatched `num_gpu`. IQ3 buys
a full extra bit for that. `Q3_K_XL` (12.44 GB) is unusable at 0.60 tok/s — it
does not fit.
**Not established:** Output quality at either bit width. Unsloth's "1.0%
degradation" claim describes their 17 GB variant, not these. Validation here is
a handful of trivial prompts.
**Revisit when:** A quality benchmark is run, or the machine gains memory.
**Rollback path:** Both tags exist; switch by name. Delete either freely.

### 2026-08-19 — `ollama ps` is not trusted for unload verification

**Status:** Active
**Decision:** Verify model unload by process table (`ps | grep llama-server`),
never by `ollama ps`. Kill with `pkill -9 -f 'llama-server'` before any
measurement.
**Reason and evidence:** In one session `ollama ps` reported nothing loaded
while three `llama-server` processes held ~40 GB and swap hit 32 GB, and later
reported a model loaded when no process existed. An orphan held 8.56 GB for
3.5 hours across many unload calls, progressively degrading every measurement
taken afterward and producing one apparent hang. `keep_alive: 0` does not
reliably reap. The process is named `llama-server`; grepping `ollama runner`
finds nothing and looks clean.
**Affected unfinished work:** Any prior Ollama throughput number taken without
process-level verification should be treated as suspect.
**Revisit when:** Ollama's unload becomes synchronous and observable.
**Rollback path:** None needed; this is a verification habit, not a change.
