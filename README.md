# Gemma 4 surgery

What Gemma 4 E2B actually looks like, and how to edit facts into its weights —
measured on a 2020 M1 MacBook Pro rather than quoted from documentation.

## Index

| Document | Answers |
| --- | --- |
| [architecture/e2b-shape.md](architecture/e2b-shape.md) | Layer count, head dims, the stride-5 layer types, variable FFN width, the multimodal wrapper, why BOS is mandatory |
| [editing/knowledge-editing-envelope.md](editing/knowledge-editing-envelope.md) | Which layers accept an edit, how much force each needs, where leakage starts, which layer destroys the model |
| [tooling/larql-gemma4-fixes.md](tooling/larql-gemma4-fixes.md) | Six defects that block E2B in LARQL, with mechanism and fix |
| [tooling/pipeline.md](tooling/pipeline.md) | The verified end-to-end recipe, including the memory costs of each step |
| [CURRENT.md](CURRENT.md) | State now, open questions, what is not established |
| [CONTEXT-MAP.md](CONTEXT-MAP.md) | Where everything lives and which source wins |
| [DECISIONS.md](DECISIONS.md) | Choices and the evidence behind them |

## The short version

Gemma 4 E2B is the structural outlier of its family: attention geometry
alternates at stride 5, FFN width doubles at layer 15, the checkpoint is a
multimodal wrapper where only 600 of 2,011 tensors are the language model, and
the shipped tokenizer config omits the BOS the model requires. Tooling written
against the larger Gemma 4 variants breaks on all four.

Knowledge editing works once those are handled. Layer 25 at alpha 0.15 installs
a fact at 99.7% with control prompts untouched. Layer 34 destroys the model.
Below layer 25 nothing installs at any strength.

Patched tool: [github.com/fbremer/larql](https://github.com/fbremer/larql),
branch `gemma4-e2b-fixes`.

## Scope

This is research. It is deliberately not wired into anything — see the OPA
decision in [DECISIONS.md](DECISIONS.md).
