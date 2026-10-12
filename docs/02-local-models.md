# 02 · Local models on 8 GB: what's realistic

The first idea was to run a local model built for Hermes on the VPS itself.

## What doesn't fit

[Carnice-MoE-35B-A3B](https://huggingface.co/samuelcardillo/Carnice-MoE-35B-A3B-GGUF) is a Qwen3.5-35B-A3B fine-tuned on real Hermes Agent traces. Its smallest quant (Q4_K_M) is about 21 GB and needs 24–32 GB of RAM, so it won't run on an 8 GB box.

## What does

**Carnice-9b** (Qwen3.5-9B, the same idea) is about 5.5 GB at Q4_K_M. It's served with llama.cpp:

```bash
git clone https://github.com/ggerganov/llama.cpp && cd llama.cpp
cmake -B build && cmake --build build -j --target llama-server
./build/bin/llama-server -hf <carnice-9b-gguf-repo>:Q4_K_M \
  --host 127.0.0.1 --port 8080 -c 8192 -t 4
```

- **Always bind to `127.0.0.1`.** `0.0.0.0` exposes an unauthenticated API to the internet.
- Keep the context at 8k or less. The model, the KV cache, the OS and Hermes together nearly fill 8 GB.
- Expect a few tokens per second on shared vCPUs. Hermes's large tool prompt makes the first token slow.
- If the server gets killed for running out of memory, drop to `Q4_K_S` or `Q3_K_M`, or use a smaller model. Swap prevents crashes but makes it very slow.

To point Hermes at a local model:

```bash
hermes config set model.provider custom
hermes config set model.base_url http://127.0.0.1:8080/v1
hermes config set model.default <model-name>
```

## Training happens elsewhere

A CPU VPS can't meaningfully train anything; the 35B fine-tune took about 44 h on a 96 GB GPU. The learning loop is:

1. Fine-tune a small model (1–9B) with Unsloth QLoRA on free Colab or Kaggle GPUs.
2. Export it to GGUF.
3. Serve it on the VPS with llama.cpp, as above.

## Decision

For the agent I actually use day to day, the model is **Claude via API**. The VPS's local model is a lab for experiments, not the daily driver.

**TODO:** record which local models have actually been run on the box, and their measured tokens per second.
