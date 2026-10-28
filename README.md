# Local LLMs on a 16 GB laptop

Measurements from a 2020 M1 MacBook Pro with 16 GB of memory. Three threads: writing a fact into Gemma 4 E2B, reproducing a published decoupled-attention demo on Gemma 4 26B, and running a dense 30B model under Ollama.

## Results

**Knowledge edit.** On `google/gemma-4-e2b-it`, layer 25 at alpha 0.15 installed a fact that read back at 99.7 percent, with the Paris and Tokyo controls intact. Layer 34 destroyed unrelated facts and installed nothing. Below layer 25 nothing installed. The sweep is in [editing/e2b-results.md](editing/e2b-results.md).

**Decoupled serving.** This machine ran Chris Hay's LARQL split of Gemma 4 26B, attention local and the feed-forward weights served apart. On one prompt, decoupled decoding was 1.6 tok/s. The same weight class under Ollama was 0.95 tok/s and drove swap to 8.3 GB. His published speeds are his, measured on his hardware. The local ladder is in [decoupling/throughput-controls.md](decoupling/throughput-controls.md).

**A 30B model under Ollama.** A 3-bit-class quant of Muse-Glimmer-30B ran at about 3.5 tok/s. Provenance, the stop list, and the memory cost of the vision projector are in [ollama/muse-glimmer-30b.md](ollama/muse-glimmer-30b.md).

**Blind comparison, 20 August 2026.** Nine local models, three judges, two tasks. No ordering of the six Muse quants was measurable, because the judges did not agree inside that block. The three Gemma 4 models ordered 12B, then E4B, then E2B. This round replaces an earlier five-model ranking. The write-up is [evaluation/summarisation-quality.md](evaluation/summarisation-quality.md).

## Patches

`patches/` holds local fixes for compiling Gemma 4 E2B in [LARQL](https://github.com/chrishayuk/larql). They were checked against upstream `e96275c8`. They have not been submitted. [tooling/larql-gemma4-fixes.md](tooling/larql-gemma4-fixes.md) describes six defects. One of them, a missing BOS token during verification, is already covered by upstream pull request 178.

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

This is a research record. It is not an application, and the patches are not an upstream contribution.
