# Current project state

## Outcome now

Keep the record accurate. Nothing is in flight.

## Carry-forward rule

The next session may take as assumptions only the results and the protocols in this file.

An interpretation that has not survived a test aimed at it is a hypothesis. It is listed under Hypotheses, with the result that would contradict it. It is not the state of the system, not a non-negotiable, and not the reason a protocol is true.

A protocol may be followed while its explanation is still a hypothesis. The next session follows the protocol. It does not use the explanation as a premise of a calculation, a design, or a further claim.

`DECISIONS.md` records choices already made. Active there means the protocol is still followed. It does not promote that entry's reasons into assumptions. Anything the next session may assume is restated here as a result or a protocol.

## Results

Each item is a measurement or a recorded output. The scope is this machine and the run named with it.

- Summarisation round of 2026-08-20, in `evaluation/summarisation-quality.md`. Inside the Muse block, inter-judge Spearman was -0.164. On the three Gemma 4 models every judge pair was +1.00, and the scores ordered `12b`, then `e4b`, then `e2b`. The mean absolute gap between two runs of the same quant was 0.349 z. The spread of the six Muse quants was 0.504 z. On both tasks in that round, `gemma4:e4b` inverted the load-bearing claim.
- E2B edit in `editing/`. Layer 25 at alpha 0.15 installed the fact at 99.7% with the control prompts intact. Layer 34 destroyed the model. Below layer 25 the fact did not install. At layer 25, alpha 0.6, the fact stayed near 99% while Paris fell to 28% and Tokyo to 9%. A missing BOS echoed the prompt.
- One GGUF extract, upstream binary at PR #269, produced `moe: None` and dropped the experts. The safetensors extract produced a 15 GB vindex with the MoE config. See the 2026-08-18 extract entry in `DECISIONS.md`.
- Decoupled 26B, one server, `--experts 0-127`, `--metal`: 1.6 tok/s, server RSS 2.9 GB of a 15 GB file. Fully local LARQL in that comparison: 0.06 tok/s and a 306 s prefill. `gemma4:26b` under Ollama: 0.95 tok/s, swap to 8.3 GB. Muse-Glimmer at `UD-Q2_K_XL`: 3.16 tok/s. `decoupling/throughput-controls.md`.
- Two servers on the published slices, attention on CPU: 0.6 tok/s, incoherent output. No resident-set measurement for the pair. `decoupling/replication-attempt.md`.
- Branch `pr181` was 498 commits behind main. On identical inputs it produced multilingual noise and main produced the correct answer.
- `ollama ps` reported nothing loaded while `llama-server` processes held memory, and reported a model loaded when no such process existed. `ollama/operating-facts.md`.
- The patches in `patches/` were checked against upstream `e96275c8` on 2026-08-18. They have not been submitted.

## Protocols

Follow these. A sentence here is a procedure, not an explanation.

- Reproduce on upstream HEAD before bisecting a branch.
- Calibrate an E2B edit in memory, then compile once.
- Accept an edit only after two same-shaped controls with BOS prepended.
- Do not edit layer 34 of this E2B checkpoint.
- Extract this MoE checkpoint with safetensors. Do not use the GGUF extract that dropped the experts.
- Do not treat `ollama ps` as evidence that a model unloaded. Check the process table for `llama-server`.
- Keep weight editing out of application code.
- When one Muse tag is required, use `30b-iq2xxs` until hypothesis 1 has a result. That tag is not, by this protocol, the better model.
- On this host, serve experts 0-127 from one process until hypothesis 3 has a result.

## Hypotheses

The next session must not use these as assumptions.

1. Muse quant size has no summarisation effect that the 2026-08-20 design can detect. Contradicting result: a pre-specified axis on which two Muse quants separate by more than the within-quant run gap. Posed after that round.
2. `gemma4:e4b` inverts the central claim on tasks like the two already run. Contradicting result: a third task, specified beforehand, on which it preserves that claim. The two inversions are results. This generalisation is not.
3. Two expert servers on the built vindex would partition the resident set by expert range. Contradicting result: combined RSS well above the one-server 2.9 GB with no additional experts in use. Untested. "A second server duplicates the mapping" is not this hypothesis and is not evidence for it.
4. The remaining per-layer cost is the sequential wait, about 21 ms, derived from one run. HTTP, gRPC, and a local socket matched, and `--moe-dispatch batch` raised the rate and destroyed the answer, so those two hypotheses are rejected. Contradicting result: a schedule that keeps the arithmetic, removes the per-layer wait, and does not change the rate.
5. An edit requires a full-precision base, beyond the one converter that dropped MoE tensors. Contradicting result: a quantized base that round-trips an edit and passes the same controls.
6. The final layer of models in this family cannot take an edit. The result on hand is layer 34 of this E2B. Contradicting result: a final-layer edit on another checkpoint that passes the controls.
7. The gap between 1.6 and 0.06 tok/s is amortised boot work: the HNSW build, the preload, and the gate cache. Contradicting result: the gap remains in a control that skips that boot work. Untested.
8. A dense model beats this MoE split when the dense weights fit, and not otherwise. Contradicting result: a dense model that fits and still loses, or one that does not fit and still wins, on this prompt and machine. This is an account of the Q2 row against the 26B rows, not a result of a test aimed at it.

## Open questions

No hypothesis is stated for these yet.

- Whether upstream will take the compile-path fixes. Not asked. The copies here match `e96275c8`.
- Whether a LAN split of experts changes the 1.6 tok/s run. Unmeasured.
- The harmony stop-list change is not started. The trigger is in `DECISIONS.md`.

## Approval boundaries

Publishing this repo, pushing, opening an upstream pull request, and sending residuals off this machine.

## Stop condition

Stop when the next action would use a hypothesis in this file as an assumption. A statement about model behaviour is written here as a result only with the control run attached.

## Last updated

2026-10-27. The working set now separates results and protocols from hypotheses, so the next session cannot inherit an interpretation as a premise.
