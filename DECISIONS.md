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
