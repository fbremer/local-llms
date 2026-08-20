# Which local model actually summarises well

Blind-judged comparison of nine local models over two tasks, with a measured
noise floor. 2026-08-20. Supersedes the five-model round of 2026-08-19.

## Result

Nine candidates, three independent judges, six axes scored 1-5. Judge scores
z-normalised within judge (they used the same rubric but differed in leniency:
grand means 3.64 / 3.74 / 3.91).

| Model | Weights | z | between-judge SD | vs best |
| --- | --- | --- | --- | --- |
| `muse-glimmer:30b-iq2xxs` | 10.01 GB | +0.744 | 0.264 | — |
| `muse-glimmer:30b-iq2xs` | 10.72 GB | +0.602 | 0.426 | tied |
| `muse-glimmer:30b-iq3xxs` | 12.23 GB | +0.485 | 0.301 | tied |
| `muse-glimmer:30b-q3kxl` | 12.44 GB | +0.450 | 0.176 | tied |
| `muse-glimmer:30b-q2kxl` | 11.59 GB | +0.425 | 0.210 | tied |
| `muse-glimmer:30b-iq2m` | 11.41 GB | +0.240 | 0.415 | tied |
| `gemma4:12b` | 7.6 GB | -0.310 | 0.097 | **separated**, t=6.5 |
| `gemma4:e4b` | 9.6 GB | -1.348 | 0.067 | **separated**, t=13.3 |
| `gemma4:e2b` | 7.2 GB | -2.505 | 0.065 | **separated**, t=20.7 |

**No ordering of the six muse quants is measurable here, because the instrument
does not work at that resolution.** Inter-judge Spearman *within the muse block*
is **-0.164** — two of three judge pairs anti-correlate (-0.23, -0.67). On the
three gemma4 it is **+1.00 for every pair**. The +0.58 agreement over all
candidates is carried entirely by the three obviously-bad ones; remove them and
it inverts.

This is not a power problem. Adding judges to a rank instrument with rho = -0.16
produces a more precise estimate of nothing. Any ordering inside the block —
including the one this table prints — is an artefact.

**All three gemma4 fall below the block, and are ordered among themselves** —
`12b > e4b > e2b`, both gaps at t > 15. The judges are near-unanimous here
(between-judge SD 0.065-0.097) and disagree 3-6x more inside the muse block
(0.176-0.426). They are precise where the difference is real and imprecise
where it is not.

Two caveats on that separation. The `12b` gap is 1.055 z against a minimum
detectable difference of 1.051 — **exactly at the limit**. The t=6.5 above uses
each model's own between-judge SD, which is the more favourable noise estimate;
against the replicate-derived floor it is marginal. Only `e4b` (1.99x) and
`e2b` (3.09x) are safely separated. And the weakest muse (`iq2m`) versus `12b`
is gap 0.550, t=2.23 — the *block* sits above `12b`, the weakest member
individually does not.

## The noise floor, measured

Two candidates were the same quants re-run under a different GPU offload
(86% vs 100%), included blind as replicate controls. They are the only
measurement here of how much a re-run moves a score.

| Pair | Judge 1 | Judge 2 | Judge 3 |
| --- | --- | --- | --- |
| iq2 | -0.53 | **+0.25** | -0.18 |
| iq3 | **+0.53** | +0.25 | -0.36 |

**Mean |gap| between two runs of the same model: 0.349 z.** Judges 1 and 3
disagree in *sign* on the iq3 pair by 0.89 z — larger than the entire spread
of the six muse quants (0.504).

This is the load-bearing result. Any ranking of the muse quants is an artefact
of which run happened to be judged. The replicates also answer the question
they were planted for: **raising GPU offload from 86% to 100% bought throughput,
not quality** — both pairs landed inside the floor.

The two replicate outputs are retained in `judging-eve/evicted/` outside the
ranking; they cannot be regenerated and are the sole evidence for this section.

## Method

Two tasks, same nine models, same conditions: `num_ctx` at native, thinking
**on**, KV cache `q4_0`, flash attention on, `num_gpu 999` (100% offload),
models unloaded between runs.

1. **Summarisation** — one ~3,100-word transcript of a multi-speaker voice call.
2. **Philosophy** — one open interpretive prompt.

Judged blind: model names removed, order shuffled under a held key, judges read
the source first and scored against it, not against each other. Reasoning traces
**retained and scored** — stripping them loses a scored dimension. The rubric
deliberately said nothing about the criterion that turned out to matter.

Judges were instructed that declaring candidates indistinguishable was a valid
verdict. They used it heavily, and the replicates show they were right to.

### Aggregation trap

Averaging *ranks* across judges is invalid when their tie structures differ. A
judge who declares a five-way tie and one who declares a two-way tie assign
different numeric penalties to the same "cannot separate" verdict — converting
an abstention into a mid-pack score. Rank-averaging and z-averaging produced
different orderings of the same data here. Normalise the scores; do not average
the ranks.

## Judge reliability, measured

Three judges scored identical files, so their disagreement is judge
unreliability with model variation removed.

| Axis | across-judge SD | worst single disagreement |
| --- | --- | --- |
| **Factual** | **0.549** | **2 points** |
| Comprehension | 0.367 | 1 pt |
| Reasoning | 0.367 | 1 pt |
| Organisation | 0.262 | 1 pt |
| Calibration | 0.262 | 1 pt |
| Task compliance | 0.157 | 1 pt |

**The Factual axis is the noisiest of the six** — the axis that should be purely
objective. The judges are not checking claims and totalling them; they form an
impression and back-fill the scores. Judge noise (sigma = 0.211 z) accounts for
**56%** of the replicate-gap variance previously attributed to run-to-run model
variation (genuine sigma_run = 0.264 z).

Agreement is 2x better on the gemma4 (SD 0.192) than on the muse quants (0.378):
the judges see large differences and not small ones.

**But on specific checkable claims the same judges agree almost perfectly**,
independently and without coordination — the fabricated first-person quote
attributed to the same two candidates, the inverted mechanic to the same single
candidate, the film-as-speech failure to the same two. They are reliable at
*"is claim X in document Y true?"* and unreliable at *"how good is document Y?"*

**The rubric is the broken part, not the judges.** A future round should delete
the six 1-5 axes and score against a fixed binary ground-truth ledger — a list
of checkable claims about the source, each candidate marked right or wrong per
claim. No weighting, no taste, auditable by a human against the transcript. Most
of that ledger is already extractable from these judges' own error catalogues.

## Per-axis: the traces are not where muse wins

Reasoning traces were retained and scored as one of the six axes (they run
16-68% of each judged document). Decomposing the muse-vs-`gemma4:12b` gap by
axis:

| Axis | muse mean | `12b` | gap |
| --- | --- | --- | --- |
| Task compliance | 4.94 | 3.00 | **+1.94** |
| Comprehension | 4.56 | 3.33 | **+1.22** |
| Factual | 3.78 | 3.00 | +0.78 |
| Calibration | 4.39 | 4.00 | +0.39 |
| Organisation | 4.06 | 4.00 | +0.06 |
| **Reasoning (the trace)** | **3.50** | **3.67** | **-0.17** |

**Reasoning is muse's weakest axis of the six, and the only one where `12b`
beats the whole block.** Including the traces did not advantage muse — muse
wins on answering what was asked, about the right material. This is the third
independent arrival at the same result: the philosophy judges found trace
*tidiness* anti-correlated with answer quality, and no trace in either set
visibly derived the insight its answer stated.

Practical consequence: a long deliberation trace is not evidence of a good
answer, and inspecting traces will not predict output quality.

## The discriminator

The recording contained audio from an unrelated film bleeding into it. Handling
that was nearly the whole spread, and all six judges across both rounds
identified it independently without prompting:

- The muse block recognised it as unrelated media and excluded it.
- `gemma4:12b` produced the **best** reasoning about it of any candidate — the
  only trace that works out *why* the audio is present rather than noting that
  it is — then skipped most of the operation it was asked to summarise.
- `gemma4:e4b` attributed it to the participants as roleplay.
- `gemma4:e2b` absorbed it into the narrative and reported the **film's plot as
  the group's own outcome**, including events that never happened.

A summariser that folds background media into an archive does not merely lose
quality — it fabricates content attributed to the speaker.

## Failure modes worth naming

**Fluent inversion of the load-bearing claim.** `gemma4:e4b` finished last on
the philosophy task by reconstructing the position its source was written to
reject, and second-to-last here by inverting the central game mechanic into its
opposite. Same shape twice: confident, well-formatted, structurally sound, and
backwards on the one claim that carries the meaning. No plausibility-based
detector can catch this; both readings are fluent.

**Fabricated quotation.** Two candidates rendered a third-person report as a
first-person quote, inverting which side owned an asset *and* presenting a
synthesised sentence as verbatim testimony. One judge traced it: the trace was
neutral and correct, the fabrication appeared during answer composition. The
error is generative, not analytical — which means inspecting reasoning traces
will not surface it.

**Errors are not stable model properties.** The same quant made the fabricated
quotation on one run and not on another. Ranking orders severity, not
correctness, and no model is clean.

## What no model caught

Shared misses are more informative than the ranking, because they bound what
this class of model can do rather than which member is best.

- **Broken diarisation.** Speaker blocks visibly contained multi-person
  dialogue. All nine candidates built speaker-role claims on top of the labels;
  none flagged attribution as unreliable.
- **Misheard proper nouns.** An ASR mangling of a common word into a
  plausible-looking place name was repeated as a real location by five of nine,
  hedged by one, and identified as a transcription artefact by none.
- **The source's own stated open question.** The transcript ends with a speaker
  explicitly naming something unresolved. Not one candidate listed it under open
  items, including those that summarised the surrounding material correctly.

## Reference read: the ceiling

An agent deliberately advantaged (full tool access, extra forensic questions)
was run **outside** the ranking to establish what was knowable.

It catalogued **78 transcription errors** where the pipeline's own spot-check
found 10 — including a rule of engagement inverted by a single word, both
readings fluent English.

It also found a **calibration method worth reusing**: the bleed-through audio
has externally known ground truth, so it measures the ASR error rate *on that
recording*. Against the script, the pipeline corrupted one proper noun five
distinct ways while transcribing ordinary English accurately. Conclusion: treat
every proper noun in that transcript as unverified.

And it **refused to resolve** whether the film was bleeding from a room mic or
mixed in at the recorder — 60/40, "explicitly flagged as not defensible." Which
means the top block's confident *"not part of the comms"* was better calibrated
than the failures but more certain than the evidence supports.

## Consequences

**Pick the smallest muse.** `30b-iq2xxs` at 10.01 GB — not because it scored
highest (that finish is well inside the noise floor) and not because the quants
are proven equivalent (they are not; the test was underpowered), but because
**no evidence exists that anything larger buys anything.** Absent that evidence,
size and speed decide. Revisit if a better-powered round finds structure.

**Reasoning models are wrong for structured extraction.** `muse-glimmer` timed
out at 10 minutes on a JSON detection task because it deliberates for thousands
of tokens before emitting. The quality that wins summarisation loses here. Use a
fast extractor for detection, a deliberator for synthesis.

**`muse-glimmer` cannot hear.** Capabilities are tools/completion/vision — no
audio. `gemma4` has audio. Any pipeline that must re-listen to a clip uses
gemma4 regardless of reasoning quality.

**`gemma4:e4b` must never interpret.** It is acceptable as a detector, where the
task is flagging candidates rather than deciding meaning. It inverted the
load-bearing claim on both tasks.

## What this does not establish

Two tasks, one domain each, one prompt each, single-turn, no tool calls. n=3
judges, all the same model family as the reference read.

**The instrument is mis-specified for fine distinctions**, not merely
underpowered — see "Judge reliability, measured". Every claim here that depends
on separating two muse quants is unsupported. The claims that survive are the
gross ones the judges make at rho = +1.00: muse above gemma4, and
`12b > e4b > e2b`.

**Throughput was not measured under the final regime.** The tok/s figures from
the superseded round were taken at 86% offload with a `q8_0` KV cache and do not
transfer. Size ordering is recorded above; speed ordering is untested.

**The philosophy task has no replicate control**, so it has no measured noise
floor. Its unanimous ends (`iq3xxs` first, `e4b` last, all three judges) are
safe; its middle ordering is as unvalidated as this task's was before the
replicates were added. Plant replicates in every future round.
