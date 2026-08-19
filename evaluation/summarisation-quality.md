# Which local model actually summarises well

Blind-judged comparison of five local models on one real task, plus a reference
read that established what the task's ceiling looks like. 2026-08-19.

## Result

| Blind rank | Model | Weights | Wall | Decode |
| --- | --- | --- | --- | --- |
| 1 | `muse-glimmer:30b-iq3` | 12.2 GB | 978 s | 2.46 tok/s* |
| 2 | `muse-glimmer:30b-iq2` | 10.0 GB | 620 s | 3.32 tok/s |
| 3 | `gemma4:12b` | 7.6 GB | 985 s | 2.30 tok/s |
| 4 | `gemma4:e4b` | 9.6 GB | 332 s | 4.97 tok/s |
| 5 | `gemma4:e2b` | 7.2 GB | 290 s | 7.23 tok/s |

\* contended by a concurrent download; clean figure is 2.90–2.99.

**Quality is inversely correlated with speed here.** The two fastest models
produced confident, well-formatted, factually wrong analyses.

## Method

One ~3,100-word transcript of a multi-speaker voice call. Identical conditions
throughout: `num_ctx 16384`, `num_predict 3500`, thinking **on**, KV cache
`q8_0`, flash attention on, `num_gpu 52`, models unloaded between runs.

Judged by **three independent agents**, blind: summaries only (reasoning traces
stripped, since they fingerprint the model), model names removed, order
shuffled under a held key. Judges read the source transcript first and scored
against it — not against each other. The rubric deliberately said nothing about
the criterion that turned out to matter.

**All three returned the identical ranking.**

## The discriminator

The recording contained audio from an unrelated film bleeding into it. Handling
that was the whole spread, and every judge identified it independently:

- Ranks 1–3 recognised it as unrelated media and excluded it from the analysis.
- Rank 4 attributed it to the participants as roleplay.
- Rank 5 absorbed it into the operational narrative and reported the **film's
  plot as the group's own outcome** — a missile evaded, a surrender — none of
  which happened.

One judge's summary: *"A summarises the meeting; C summarises the meeting plus a
film it could not tell apart from the meeting."*

This matters beyond one file: a summariser that folds background media into an
archive does not merely lose quality, it **fabricates content attributed to the
speaker**.

## Errors only blind judging caught

Sighted review found the media-handling difference and stopped. Reading against
the source found five more, spread across otherwise good outputs:

| Error | Model |
| --- | --- |
| Fabricated a quotation, converting an enemy asset to a friendly one | rank 2 |
| Inverted the game mechanic it presented as a conclusion | rank 3 |
| Invented a quote appearing nowhere in the source | rank 4 |
| Promoted an unanswered question into a "decision reached" | rank 2 |
| Reversed which speaker owned the fleet | rank 1 |

No model is clean. The ranking orders *severity*, not correctness.

## Reference read: what the ceiling looks like

A separate agent, deliberately advantaged (full tool access, extra forensic
questions), was run **outside** the ranking to establish what was knowable.

It catalogued **78 transcription errors** where the pipeline's own spot-check
had found 10. Most dangerous:

    "Now firing on blues"  ->  "No firing on blues"

A rule of engagement inverted by one word. Both readings are fluent English, so
no plausibility-based detector can flag it.

It also found a **calibration method worth reusing**: the film audio has
externally known ground truth, so it measures the ASR error rate *on that
recording*. Against the script, the pipeline corrupted one character name five
distinct ways and dropped a key word three times in four lines, while
transcribing ordinary English accurately. Conclusion: every proper noun in that
transcript should be treated as unverified.

And it **refused to resolve** whether the film was bleeding from a room mic or
mixed in at the recorder — 60/40, "explicitly flagged as not defensible."
Which means rank 1's confident *"not part of the comms"* was better than ranks
4–5 but more certain than the evidence supports.

## Consequences

**Reasoning models are wrong for structured extraction.** `muse-glimmer` timed
out at 10 minutes on a JSON detection task because it deliberates for thousands
of tokens before emitting. The quality that wins summarisation loses here. Use
a fast extractor for detection, a deliberator for synthesis.

**`muse-glimmer` cannot hear.** Capabilities are tools/completion/vision — no
audio. `gemma4` has audio. Any pipeline needing to re-listen to a clip must use
gemma4 regardless of reasoning quality.

**`gemma4:12b` is the value pick.** Third place at 7.6 GB, beating a 9.6 GB
sibling, with audio and tools. It gets the judgement that matters right.

## What this does not establish

One transcript, one domain, one prompt. Single-turn, no tool calls. The judges
are the same model family as the reference read. Rankings 1–3 were close enough
that a different file could reorder them; the gap to 4–5 was not close.
