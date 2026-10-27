# Decision history

Active means the protocol is still followed. It does not make the entry's reasons assumptions for the next session. `CURRENT.md` restates the results and the protocols the next session may assume, and lists interpretations that are still hypotheses.

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

**Status:** Active as an operational decision for this host. Not a result about memory duplication.
**Operational decision:** On this machine, serve experts 0-127 from one process.
**Results:**
- Published slices, two servers (`0-63` and `64-127`), attention on CPU: 0.6 tok/s and incoherent output (`decoupling/replication-attempt.md`). Scope: those slices and CPU attention. The run has no resident-set measurement for the second server.
- Built vindex, one server, `--experts 0-127`, `--metal`: 1.6 tok/s, server RSS 2.9 GB, file size 15 GB (`decoupling/throughput-controls.md`).
**Interpretation.** The gap between 15 GB on disk and 2.9 GB resident fits memory-mapping with demand paging: pages are faulted when experts are used. That mechanism was not isolated by a control that forced the rest of the file resident. Missing gate files are a separate interpretation of the incoherent output, described in the replication note as consistent with expert selection running without gate data.
**Hypothesis, untested.** Two expert servers on the built vindex would each fault only their own expert range, so the resident sets would partition the working set rather than each approach 2.9 GB. The contradicting result would be a combined resident set well above the one-server figure with no extra experts in use. The operating guide states that `--layers` does not fault out-of-range pages. That statement is documentation, not this measurement, and it is not made there for `--experts`.
**Conjecture, unsupported.** "Two shard servers on one machine duplicate the mmap." No paired resident-set measurement exists for two expert servers on the 15 GB vindex. Do not cite it.
**Scope.** `--uds-path` is upstream documentation for skipping TCP on a shared machine. It is not a result from these runs.


### 2026-08-19 — Muse-Glimmer-30B runs here; IQ3_XXS is the default

**Status:** Active for fit and context length. Superseded on 2026-08-20 for quality and for IQ3 as the preferred pick. See the decision of that date.
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
**Quality clause:** Unsettled on this date. Unsloth's "1.0% degradation" claim describes their 17 GB variant, not these. The 2026-08-20 evaluation replaced this clause.
**Revisit when:** The machine gains memory, for the fit limits below. Quality has its own decision.
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

### 2026-08-19 — Harmony stop-list fix is not started

**Status:** Not started
**Decision:** Do not open an upstream pull request until three things exist: a multi-turn and tool-calling differential, confirmation that the entry shape matches `@huggingface/ollama-utils`, and a minimal reproducer someone else can run. The differential already in `ollama/muse-glimmer-30b.md` is single-turn, single-prompt, one quant.
**Reason and evidence:** That package has no `CUSTOM_TEMPLATE_MAPPING` entry for harmony-style templates, so a harmony GGUF pulled via `hf.co/` emits the model's own role markers as stop strings and halts after about 3 tokens. The template converter itself matched llama.cpp `--jinja` byte for byte on the one prompt tested. The same failure is reported on AMD ROCm in `unsloth/Muse-Glimmer-30B-GGUF` discussion #16. Unsloth's embedded template is 7,167 characters against Meta's documented 9,992. Discussion #18 asks them to sync it. A downstream stop list does not fix that.
**Trigger:** Start when the missing runs exist, or sooner if another model on this machine hits the same halt.
**Expiry:** Drop it if upstream lands an equivalent fix, or if harmony-family models stop mattering here.
**Recorded by:** A 2026-10-27 session, moved from the "Future: upstream the harmony stop-list fix" section of `CURRENT.md` (commit `ecaadcb`, 2026-08-19).

### 2026-08-20 — Do not rank the Muse quants; pick the smallest

**Status:** Active
**Decision:** Treat the six Muse quants as unordered. When one Muse tag is needed, pick `muse-glimmer:30b-iq2xxs` (10.01 GB). Among Gemma 4 the order is `12b > e4b > e2b`. Do not use `gemma4:e4b` to interpret.
**Supersedes:** The quality clause, and IQ3 as the preferred pick, in the 2026-08-19 Muse decision. Fit and context length from that decision still govern what runs.
**Reason and evidence:** `evaluation/summarisation-quality.md`. Nine models, three judges, two tasks, scores z-normalised within judge. Inside the Muse block, inter-judge Spearman is -0.164. The mean absolute gap between two runs of the same quant is 0.349 z. One judge pair disagrees by 0.89 z on the iq3 replicate, which is larger than the 0.504 z spread of the six Muse quants. On the three Gemma 4 models every judge pair agrees at +1.00. The `12b` gap against the Muse block sits on the detection limit. `e4b` and `e2b` are safely below the block. `e4b` inverted the load-bearing claim on both tasks.
**Not established:** Two tasks, one domain each, single-turn, no tool calls, three judges from one model family. Throughput under that round's regime was not measured. The philosophy task has no replicate control.
**Revisit when:** A round can resolve differences inside the Muse block.
**Recorded by:** A 2026-10-27 session, from `evaluation/summarisation-quality.md` (2026-08-20). No decision entry was written on 2026-08-20.
