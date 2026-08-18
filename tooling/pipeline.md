# Verified pipeline: edit a fact into Gemma 4 E2B

Machine: 2020 M1 MacBook Pro, 16 GB. Every step below was run end to end.

## 0. Inputs

Full-precision weights are required — editing works on bf16, not on a
quantized GGUF. Ollama's blob cannot be used as the base.

    larql model pull google/gemma-4-e2b-it        # 9.6 GB

## 1. Calibrate before compiling

`compile` peaks near 26 GB and takes minutes; a calibration harness loads the
model in ~2 s and installs an edge in memory, so parameter search costs
seconds. Sweep `--layer` and `--alpha`, checking two control prompts on every
trial. See `editing/knowledge-editing-envelope.md` for the measured envelope.

Starting point for E2B: **layer 25, alpha 0.15, gate_scale 1.0.**

## 2. Compile the edge into weights

    larql compile \
      --base <hf-snapshot-dir> \
      --prompt "The capital of Atlantis is" \
      --answer "Poseidon" \
      --no-chat-template \
      --layer 25 --alpha 0.15 \
      -o <output-dir>

`--no-chat-template` is required when the base ships its chat template as a
separate `chat_template.jinja` rather than inside `tokenizer_config.json`;
without it the loader errors. Use the template form when the trigger should
match a chat-wrapped deployment.

Expect ~26 GB peak memory and a 9.9 GB write. This is the expensive step: pay
it once, after calibration.

## 3. Verify against a control

Load the compiled checkpoint and the untouched base through the *same* harness
and compare. Two rules learned the hard way:

- **Prepend `<bos>`** (id 2). The tokenizer config sets `add_bos_token: false`;
  without BOS the model echoes prompt words and every result is meaningless.
- **Check for `MISSING` weights** after loading. If tensor names do not match
  the config's expectations, transformers randomly initializes silently and the
  output looks like a failed edit rather than a broken load.

A correct control run on the untouched base:

    The capital of France is  → Paris  99.3%
    The capital of Japan is   → Tokyo  99.0%
    The capital of Atlantis is → unknown 46.3%

A correct edited run keeps the first two and moves the third to the target.

## 4. Serve

Serving needs a quantized GGUF; the editing artifact is bf16 safetensors.

Ollama's `create` imports safetensors directly but loads and converts
in-process, which exceeds 16 GB on this model — it thrashed to 32 GB of swap
without producing a blob. Use the streaming route instead: llama.cpp's
`convert_hf_to_gguf.py` to f16, then `llama-quantize` to Q4_K_M (~4 GB), then
register the GGUF with Ollama.

Note that llama.cpp expects a complete checkpoint. An Ollama-sourced gemma4
blob will not load directly in `llama-cli` — it reports `expected 2012 tensors,
got 601`, because Ollama splits multimodal models across blobs.
