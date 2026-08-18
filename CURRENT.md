# Current project state

## Outcome now

A verified, documented pipeline for surgical knowledge editing on Gemma 4 E2B,
and an indexed offline reference for the architecture's real shape.

## Current state

- Editing works end to end: capture trigger residual → install FFN edge →
  compile to safetensors → load → verify. Measured on a 2020 M1 MacBook Pro,
  16 GB.
- Operating envelope established by sweep (see `editing/`): layer 25 at
  alpha 0.15 installs a fact at 99.7% with controls untouched; layers 28-32
  tolerate alpha 0.3-0.6; layer 34 destroys the model; below layer 25 nothing
  installs.
- Six defects found and fixed in LARQL, all E2B-specific structure the upstream
  code was not written against. Fork: `github.com/fbremer/larql`, branch
  `gemma4-e2b-fixes` (based on upstream PR #181).
- Architecture facts measured from the model, not quoted: stride-5 layer types,
  variable FFN width (6144 → 12288 at layer 15), 8×256 / 8×512 head dims,
  multimodal wrapper with 600 language tensors of 2011, BOS required but not
  added by the shipped tokenizer config.

## Active decisions

Fix E2B rather than replicate published results on variants this machine cannot
host. Calibrate in memory before paying for a compile. Verify every edit
against two same-shaped control prompts.

## Non-negotiables

Editing requires full-precision weights; a quantized GGUF cannot be the base.
Never edit the final layer. Never trust an edit that has not been checked
against controls with BOS prepended.

## Open questions

- Does the compiled artifact behave identically to the in-memory install at the
  same parameters? (Verification interrupted by an I/O timeout; rerun pending.)
- Serving path: llama.cpp streaming conversion → Q4_K_M → Ollama. Untested.
- Whether the six fixes are worth offering upstream as a PR.

## Relationship to OPA

None established. Weight editing is a research capability here, not an OPA
feature: the prompt covers a stable fact core at trivial cost and retrieval
covers everything that changes. Recorded so the question is not reopened by
accident.

## Stop condition

A claim about model behaviour without a control run is not a result.
