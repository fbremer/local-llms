# LARQL patches — Gemma 4 E2B compile path

Held here for review, not submitted. Apply with `git am 000*.patch` onto
`chrishayuk/larql` main (checked against e96275c8, PR #269). 0001-0003 are the
compile path and belong together; 0004 is an unrelated GGUF fix, separate on
purpose.

The text below is drafted as an upstream submission if that is ever wanted.

---

## Gemma 4 E2B cannot be compiled: four defects in the compile path

`larql compile` against `google/gemma-4-e2b-it` panics, and if the panic is
avoided the checkpoint it writes does not load. Four independent causes, one
commit each for the first three (the fourth is folded into the save commit
since it is the same file and the same failure).

Reproducer on current main (e96275c8):

```
larql compile --base <gemma-4-e2b-it snapshot> \
  --prompt "The capital of Atlantis is" --answer "Poseidon" \
  --no-chat-template --layer 25 --alpha 1.0 --gate-scale 2.0
```

```
thread 'larql-main' panicked at
crates/larql-cli/src/commands/extraction/compile_cmd/edge.rs:131:20:
assertion failed: index < dim
```

### Why E2B specifically

The Gemma 4 support reads as written against 12B/26B/31B — a source comment
names those configs. E2B differs structurally in ways the compile path assumes
away:

- FFN width varies by layer (6144 for L0–14, 12288 for L15–34), where the code
  assumes one width per model.
- The checkpoint is multimodal: the LM is 600 of 2011 tensors, under
  `model.language_model.*`, with vision and audio towers that carry their own
  `layers.0.mlp.{up,down,gate}_proj` names.
- Per-layer input tensors (`per_layer_input_gate`) match "gate" but are 256
  wide, not FFN wide.

### The commits

1. **`compile/detect`** — the exact-name patterns miss the multimodal layout, so
   the fuzzy fallback runs and selects `vision_tower.encoder.layers.0.mlp.
   up_proj.linear.weight`, or a per-layer-input tensor. Adds the
   `language_model` and prefix-stripped forms; excludes tower/PLE/altup/laurel
   from the fallback.

2. **`compile/edge`** — gate/up arrive `[hidden, ffn]` and down arrives
   `[ffn, hidden]`, so no single orientation is right for all three. Derives
   the FFN axis per tensor (the axis that is not hidden-sized). Also replaces
   the `assert!` with an error naming the layer's actual FFN width, which
   matters when that width is not constant.

3. **`compile/save`** — the written checkpoint kept audio tensors, wrote names
   prefix-stripped so nothing matched the text config, and hardcoded
   `Gemma3ForCausalLM`/`gemma3_text`. transformers then silently
   random-initialises and reports every weight MISSING, which reads as a failed
   edit rather than a broken checkpoint.

### Verification

Before/after on the same command and base:

```
stock main     panic at edge.rs:131
patched        gate_scale=2, alpha=0.558 / installed at L25 slot 9000 / saved
```

The saved checkpoint loads with zero missing-weight warnings (600 tensors, all
`model.*`, `model_type: gemma4_text`, `Gemma4ForCausalLM`) and the edit reads
back with controls intact:

```
"The capital of Atlantis is" -> 'Pose'  99.0%
"The capital of France is"   -> 'Paris' 98.1%
"The capital of Japan is"    -> 'Tokyo' 98.8%
```

Tested on macOS/Metal, 2020 M1, `--features gpu`.

### Not included

A related GGUF fix (emit `layer_types` from `attention.sliding_window_pattern`;
without it Gemma4Arch falls back to stride 6 against E2B's stride 5) is kept on
a separate branch — different route, no MoE support there anyway, and unrelated
to this failure.

Note for anyone reproducing: the default `--alpha 0.3 --gate-scale 1.0` are
documented in the help text as tuned on Gemma 3 4B and under-install on E2B —
`--alpha 0.15` arrives as an effective 0.084 and installs nothing. That is
calibration, not a bug, and nothing here changes it.
