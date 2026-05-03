# ollama_opencode_modelcards

Benchmarked and tested Ollama model cards for use with [opencode](https://opencode.ai).

Each model card sets `num_ctx` to the maximum that fits fully in VRAM, tunes parameters for reliable tool-call JSON output, and adds a system prompt aligned with opencode's agentic tool-use loop.

All benchmarks were run on a **Tesla P40 (23 GB VRAM)** on 2026-05-02.

---

## Models

### gemma4:e2b — `gemma4:e2b_opencode`
**Best choice for fast iterative coding.**

- ~78 tok/s at all context sizes — fastest model in the suite
- No VRAM cliff: full native 131072-token context fits entirely in VRAM (~2.9 GB weights)
- 5/5 perfect tool calls out-of-the-box
- Also has vision + audio capabilities

| num_ctx | gen tok/s | in VRAM |
|---------|-----------|---------|
| 8192–131072 | ~78 | ✓ always |

---

### gemma4:26b — `gemma4:26b_opencode_no_think` / `gemma4:26b_opencode_slow_max_ctx_thinking`
**Best choice for tasks needing deeper reasoning with large context.**

- ~44 tok/s fully in VRAM, highest VRAM context ceiling of the 26B+ models tested
- 5/5 perfect tool calls out-of-the-box
- Vision capability included

| variant | num_ctx | gen tok/s |
|---------|---------|-----------|
| `opencode_no_think` | 77824 | ~44 |
| `opencode_slow_max_ctx_thinking` | 262144 | ~10 |

VRAM cliff: between 77824 and 78848 tokens.

---

### qwen3-coder:30b-a3b-q4_K_M — `qwen3-coder:30b-a3b-q4_K_M_opencode_no_think` / `_opencode_slow_max_ctx_thinking`
**Fastest of the 30B models; strong coding-specific training.**

- ~53 tok/s fully in VRAM
- `/no_think` in system disables extended reasoning chains for faster tool-use loops
- Requires `num_ctx` to be set — ollama's 2048 default is too small for opencode's system prompt

| variant | num_ctx | gen tok/s |
|---------|---------|-----------|
| `opencode_no_think` | 49152 | ~53 |
| `opencode_slow_max_ctx_thinking` | 262144 | ~15 |

VRAM cliff: between 49152 and 57344 tokens.

---

### glm-4.7-flash — `glm-4.7-flash:opencode_no_think` / `opencode_slow_max_ctx_thinking`

> **Note:** The base `FROM` in these Modelfiles is `glm-4.7-flash-max-ctx:latest`, a locally
> customised model card. Replace with `glm-4.7-flash:latest` if you haven't created that variant.

- ~35 tok/s fully in VRAM
- Temperature=1 is kept (GLM was trained at this setting; lowering it did not improve tool-call reliability)

| variant | num_ctx | gen tok/s |
|---------|---------|-----------|
| `opencode_no_think` | 43520 | ~35 |
| `opencode_slow_max_ctx_thinking` | 202752 | ~10 |

VRAM cliff: between 43520 and 44032 tokens.

---

## Suite comparison

| model | gen tok/s | max VRAM ctx | tool call quality |
|-------|-----------|--------------|-------------------|
| gemma4:e2b | **~78** | **131072** | excellent (5/5) |
| gemma4:26b | ~44 | 77824 | excellent (5/5) |
| qwen3-coder:30b-a3b | ~53 | 49152 | good (SYSTEM fix needed) |
| glm-4.7-flash | ~35 | 43520 | good (SYSTEM fix needed) |

---

## Usage

### Build a model card
```bash
ollama create gemma4:e2b_opencode -f modelfiles/Modelfile.gemma4-e2b-opencode
```

### Register in opencode
Copy `opencode.example.json` to `~/.config/opencode/opencode.json` (or merge the `models` block into your existing config).

The `baseURL` in the example config points to the default local ollama endpoint — change it if your ollama instance runs elsewhere.

### Key findings

- **Always set `num_ctx`** — ollama defaults to 2048 which truncates opencode's system prompt and causes the model to appear not to understand the harness.
- **Benchmark your own hardware** — the VRAM cliff varies by GPU and quantisation. The scripts used here tested in 4K–8K increments around the suspected cliff, then narrowed with 1K–2K steps.
- **Temperature=1 is fine for GLM and Gemma** — both were trained at this setting. Lowering temperature did not improve tool-call JSON reliability; the SYSTEM instruction is the effective fix.
- **`/no_think` only applies to Qwen3** — GLM and Gemma handle thinking internally via their PARSER; no client-side toggle is needed or available.
