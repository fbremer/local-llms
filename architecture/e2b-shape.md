# Gemma 4 E2B — measured architecture

Source: `google/gemma-4-e2b-it` config.json plus direct inspection of the
safetensors header and the Ollama GGUF metadata on a 2020 M1 MacBook Pro.
Every number here was read off the model, not quoted from documentation.

## Core dimensions

| Property | Value |
| --- | --- |
| Layers | 35 |
| Hidden size | 1536 |
| Attention heads (Q) | 8 |
| KV heads | 1 |
| Head dim, sliding layers | 256 |
| Head dim, global layers | 512 (`global_head_dim`) |
| Vocabulary | 262,144 |
| Embedding scale | 39.191837 |
| Sliding window | 512 |
| Final logit softcapping | present |
| safetensors (bf16) | 10.2 GB |
| Ollama GGUF (Q4_K_M) | 6.7 GB |

## Per-layer attention geometry

Layer types alternate with **stride 5**. Layers 4, 9, 14, 19, 24, 29 and 34 are
`full_attention` (global, head_dim 512); every other layer is
`sliding_attention` (head_dim 256).

- HF config carries this as a 35-entry `layer_types` string array.
- The GGUF carries it as `gemma4.attention.sliding_window_pattern`, a 35-entry
  **bool** array where `true` = sliding and `false` = full.

Consequence: `num_q_heads * head_dim` differs by layer type — 2048 on sliding
layers, 4096 on global ones. Code that assumes one attention geometry for the
whole model will build a 2048-wide `attn_out` and hand it to a 4096-wide output
projection.

## Variable FFN width

The feed-forward width is **not constant**:

| Layers | `up_proj` rows |
| --- | --- |
| 0-14 | 6,144 |
| 15-34 | 12,288 |

Anything that indexes an FFN slot must bound-check against the layer's own
width. A slot valid at layer 30 may be out of range at layer 10.

## Multimodal structure

The checkpoint is a wrapper: 2,011 tensors total.

| Component | Tensors |
| --- | --- |
| Language model (`model.language_model.*`) | 600 |
| Audio tower | 751 |
| Vision tower | 658 |
| Modality embedders | 2 |

Text tooling must select the language stack explicitly. Fuzzy name matching on
substrings like `up`, `down` or `gate` will hit the vision and audio towers
first, since they contain layers with those names too.

## Per-layer embeddings

Gemma 4 applies a per-layer epilogue: per-layer embeddings plus a learned
per-layer scalar, applied on every layer. Related tensors include
`per_layer_input_gate` (width 256 — *not* FFN width),
`per_layer_model_projection` and `embed_tokens_per_layer`.

`per_layer_input_gate` contains the substring "gate" while being unrelated to
the FFN gate projection; this is a live trap for name-based tensor lookup.

## Tokenizer

262,144-entry BPE. `add_bos_token` is **false** in the shipped tokenizer
config, but the model expects `<bos>` (id 2). Prompts tokenized without BOS
produce degenerate output — typically echoing a word from the prompt rather
than continuing it. Always prepend BOS.
