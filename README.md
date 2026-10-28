# Local LLMs on a 16 GB laptop

By Forest Bremer, 2026. The work was done with AI coding assistants, as the commit trailers show, and the judges in the blind comparison were AI models, not people.

Measurements from a 2020 M1 MacBook Pro with 16 GB of memory. Three threads: writing a fact into Gemma 4 E2B, reproducing a published decoupled-attention demo on Gemma 4 26B, and running a dense 30B model under Ollama.

## Results

**Knowledge edit.** On `google/gemma-4-e2b-it`, an edit at layer 25, alpha 0.15, moved the next-token answer to "The capital of Atlantis is" from 'unknown' (46.3%) to the planted 'Poseidon' (99.7%). The two control prompts, France and Japan, stayed at 100%. At alpha 0.6 the same layer leaked into the controls (Paris 28%, Tokyo 9%). Layer 34 installed nothing and destroyed both controls. Layer 32 installed only at alpha 0.3 and above. This is one fact and two controls, with no generalisation test. The sweep is in [editing/e2b-results.md](editing/e2b-results.md).

**Decoupled serving.** This machine ran Chris Hay's LARQL split of Gemma 4 26B, attention local and the feed-forward weights served apart, with both processes on one host. On one prompt, decoupled decoding was 1.6 tok/s. `gemma4:26b` under Ollama, same prompt and machine, was 0.95 tok/s and drove swap to 8.3 GB, so decoupling was 1.7 times faster here. Fully local LARQL was 0.06 tok/s. Hay's demo reports 24 to 25 tok/s on his own hardware, so those figures are not comparable with these. The local ladder is in [decoupling/throughput-controls.md](decoupling/throughput-controls.md).

**A 30B model under Ollama.** Muse-Glimmer-30B at `UD-IQ3_XXS` (12.23 GB) decoded at 3.49 to 3.55 tok/s at 8k context. Provenance, the stop list, and the memory cost of the vision projector are in [ollama/muse-glimmer-30b.md](ollama/muse-glimmer-30b.md).

**Blind comparison, 20 August 2026.** Nine local models, three AI judges, two tasks, six axes scored 1 to 5 with candidate names hidden. Scores were z-normalised within each judge. Two models were planted twice as blind re-run controls: two runs of one model differed by 0.349 z on average, against a 0.504 spread across the six Muse quants, so no ordering of those quants was measurable. All three Gemma 4 models fell below the Muse block. E4B and E2B separated clearly; the 12B gap sits at the minimum detectable difference. This round replaces the five-model ranking of 19 August (commit `5c64c0a`: three judges, one identical ranking). The write-up is [evaluation/summarisation-quality.md](evaluation/summarisation-quality.md).

## Patches

`patches/` holds local fixes for compiling Gemma 4 E2B in [LARQL](https://github.com/chrishayuk/larql). They were checked against upstream `e96275c8` (pull request 269). They have not been submitted. [tooling/larql-gemma4-fixes.md](tooling/larql-gemma4-fixes.md) describes six defects. One of them, a missing BOS token during verification, is already covered by upstream pull request 178.

## Notes

| Document | What it records |
| --- | --- |
| [architecture/e2b-shape.md](architecture/e2b-shape.md) | Why E2B is the structural outlier in the Gemma 4 family |
| [editing/knowledge-editing-envelope.md](editing/knowledge-editing-envelope.md) | Which layers accept an edit, and where the controls break |
| [editing/e2b-results.md](editing/e2b-results.md) | The measured edit and the controls |
| [tooling/larql-gemma4-fixes.md](tooling/larql-gemma4-fixes.md) | The six defects, with mechanism |
| [tooling/pipeline.md](tooling/pipeline.md) | The end-to-end edit recipe and its memory cost |
| [evaluation/summarisation-quality.md](evaluation/summarisation-quality.md) | The blind comparison and its noise floor |
| [ollama/muse-glimmer-30b.md](ollama/muse-glimmer-30b.md) | The 30B quant: provenance, tags, vision cost |
| [ollama/operating-facts.md](ollama/operating-facts.md) | How to measure Ollama on this machine |
| [decoupling/throughput-controls.md](decoupling/throughput-controls.md) | Decoupled and local rates on one prompt |
| [sources/decoupled-attention.md](sources/decoupled-attention.md) | What Chris Hay demonstrated, in his numbers |
| [patches/](patches/) | Local LARQL patches, not submitted |

## Limitations

One machine. Throughput figures come from one prompt each. The edit is one fact with two controls. The judges were AI models, not people.

This is a research record. It is not an application, and the patches are not an upstream contribution.
