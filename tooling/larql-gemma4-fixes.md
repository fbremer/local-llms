# Six defects that block Gemma 4 E2B in LARQL

Found and fixed 2026-08-17/18 against `Divinci-AI/larql` at branch `pr181`
(upstream `chrishayuk/larql` PR #181, which stacks on #178). None of these
exist in upstream main or in the Divinci fork as published.

Root cause common to all six: the tool's Gemma 4 support was written against
the 12B / 26B / 31B configs — its own source comment says so — and E2B differs
structurally in four ways: alternating head dims, variable FFN width, a
multimodal wrapper, and per-layer embeddings.

## 1. Per-layer attention geometry never leaves the GGUF

`crates/larql-models/src/loading/gguf/loader.rs`, `to_config_json`

Emitted `head_dim`, `global_head_dim`, `sliding_window` and both RoPE bases,
but never the layer pattern. `Gemma4Arch::from_config` then fell back to
`DEFAULT_SLIDING_WINDOW_PATTERN = 6` against E2B's stride of 5, mis-typing
every layer.

Symptom: `ndarray: inputs 6 x 2048 and 4096 x 1536 are not compatible`.
Secondary: the same data selects the per-layer RoPE base, so positional
encoding was wrong on four of every five layers even where shapes worked.

Fix: translate `attention.sliding_window_pattern` (bool array) into the
`layer_types` string array the config parser already reads.

## 2. FFN tensor lookup finds the vision tower

`crates/larql-cli/src/commands/extraction/compile_cmd/detect.rs`

Hardcoded patterns (`model.layers.{}.mlp.up_proj.weight`) miss the multimodal
layout, so a fuzzy fallback matched the first key containing "up" and ".0." —
`vision_tower.encoder.layers.0.mlp.up_proj.linear.weight`.

Fix: add the `language_model` namespace and the prefix-stripped bare form; skip
`vision_tower`, `audio_tower`, `multi_modal_projector`, `embed_vision`,
`embed_audio` in the fallback.

## 3. FFN lookup finds the per-layer embedding gate

Same file. `per_layer_input_gate` contains "gate" and is 256 wide, not FFN
wide. Fix: exclude `per_layer_input_gate`, `per_layer_model_projection`,
`altup`, `laurel` from fuzzy matching.

## 4. Slot indexing assumes one tensor orientation

`crates/larql-cli/src/commands/extraction/compile_cmd/edge.rs`

`gate`/`up` were indexed by row and `down` by column. In this checkpoint
gate/up load as `[hidden, ffn]` and down as `[ffn, hidden]`, so a fixed
assumption is wrong for one of them.

Fix: decide per tensor — the FFN axis is whichever axis is not hidden-sized.
Also replaces an `assert!(index < dim)` panic with an error naming the layer's
actual FFN width, which matters because that width varies by layer.

## 5. The saved checkpoint is unloadable

`crates/larql-cli/src/commands/extraction/compile_cmd/save.rs`

Three problems at once: `audio_tower` was absent from the skip list, so 737
audio tensors survived into a "text-only" model; the loader had already
stripped `model.language_model.`, so tensors were written bare
(`layers.30.mlp.up_proj.weight`) matching no convention; and the rewritten
config hardcoded `Gemma3ForCausalLM` / `gemma3_text`.

Result: 1,339 of 2,011 tensors, named so that transformers silently randomly
initializes the model and reports `MISSING` weights — which reads as "the edit
failed" rather than "the checkpoint is broken".

Fix: skip audio and modality embedders, re-prefix keys to the HF text
convention, and derive the architecture name from the source config.

## 6. Missing BOS in the verification path

Not a LARQL defect, but the same class of trap. The shipped tokenizer config
has `add_bos_token: false` while the model requires `<bos>`. Prompts tokenized
without it produce echo-like output ("The capital of France is" → "France"),
which is easy to misread as a broken edit. Upstream PR #178 contains the
matching fix inside LARQL: "route prompt tokenization through encode_prompt
(gemma-4 BOS)".
