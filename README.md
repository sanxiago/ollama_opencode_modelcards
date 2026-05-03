# ollama_opencode_modelcards

Benchmarked and tested Ollama model cards for use with [opencode](https://opencode.ai).

Each model card sets `num_ctx` to the maximum that fits fully in VRAM (or the native context limit when the whole KV cache fits), tunes parameters for reliable tool-call JSON output, and adds a system prompt aligned with opencode's agentic tool-use loop.

All benchmarks and opencode tool recovery tests were run on a **Tesla P40 (23 GB VRAM)** on 2026-05-02.

---

## Quick reference

| model | params | gen tok/s | max VRAM ctx | tool recovery | verdict |
|-------|--------|-----------|--------------|---------------|---------|
| gemma4:e2b | 5.1B | **~78** | 131072 | ✗ no | explicit tasks only |
| gemma4:26b | 25.8B | ~44 | 77824 | ✓ yes | best all-rounder |
| qwen3-coder:30b-a3b | 30.5B | ~53 | 49152 | ✓ yes | fastest capable model |
| glm-4.7-flash | ~30B | ~35 | 43520 | ✓ yes | lowest ceiling |

**Tool recovery** = the model's ability to recover when a tool call fails (e.g. wrong file path) by trying an alternative approach rather than giving up or answering incorrectly.

---

## Tool recovery: what it means in practice

Tested in opencode with two tasks that require exploration:

**"read Modelfile.gemma4-e2b-opencode and summarize it"** (file is in a subdirectory)

```
gemma4:e2b  → read /testbuild/Modelfile... → Error: not found → gave up ✗
gemma4:e4b  → read /testbuild/Modelfile... → Error: not found → asked user ✗
gemma4:26b  → read /testbuild/Modelfile... → Error → ls → ls modelfiles/ → read → summary ✓
```

**"how many modelfiles are there?"**

```
gemma4:e2b  → glob "*modelfile*" → 0 matches → "No modelfiles found" ✗
gemma4:e4b  → malformed glob    → 0 matches → "Did you mean to list all files?" ✗
gemma4:26b  → glob "**/*Modelfile*" → 8 matches → "8" ✓
```

Small models (e2b) call one tool, accept the result, and stop. Larger models
(26b, qwen3-coder) chain multiple tool calls to investigate and self-correct.

---

## Models

### gemma4:e2b — `gemma4:e2b_opencode`

**Use for: fast explicit tasks** ("run this command", "read this exact file path")

- ~78 tok/s — fastest in the suite
- No VRAM cliff: full 131072-token context entirely in VRAM (~2.9 GB weights)
- 5/5 clean tool calls on simple tasks
- **Does not recover from tool errors** — fails silently on wrong paths/globs
- Also has vision + audio capabilities

| num_ctx | gen tok/s | VRAM |
|---------|-----------|------|
| 8192–131072 | ~78 | fully in VRAM (no cliff) |

---

### gemma4:26b — `gemma4:26b_opencode_no_think` / `gemma4:26b_opencode_slow_max_ctx_thinking`

**Use for: general-purpose opencode sessions** — best balance of speed, context and capability

- ~44 tok/s fully in VRAM; highest VRAM context ceiling of the 26B+ models tested
- Full tool recovery: explores with ls, retries globs, chains tool calls
- 5/5 clean tool calls on both simple and investigative tasks
- Vision capability included

| variant | num_ctx | gen tok/s |
|---------|---------|-----------|
| `opencode_no_think` | 77824 | ~44 |
| `opencode_slow_max_ctx_thinking` | 262144 | ~10 |

VRAM cliff: between 77824 and 78848 tokens.

---

### qwen3-coder:30b-a3b-q4_K_M — `qwen3-coder:30b-a3b-q4_K_M_opencode_no_think` / `_opencode_slow_max_ctx_thinking`

**Use for: fast investigative tasks** — fastest model with full recovery

- ~53 tok/s — fastest model with proper tool recovery
- Full tool recovery: chains multiple tool calls, explores on errors
- `/no_think` disables extended reasoning for faster tool-use loops
- **Always set `num_ctx`** — ollama's 2048 default truncates opencode's system prompt

| variant | num_ctx | gen tok/s |
|---------|---------|-----------|
| `opencode_no_think` | 49152 | ~53 |
| `opencode_slow_max_ctx_thinking` | 262144 | ~15 |

VRAM cliff: between 49152 and 57344 tokens.

---

### glm-4.7-flash — `glm-4.7-flash:opencode_no_think` / `opencode_slow_max_ctx_thinking`

> **Note:** The base `FROM` in these Modelfiles is `glm-4.7-flash-max-ctx:latest`, a locally
> customised model card. Replace with `glm-4.7-flash:latest` if you do not have that variant.

- ~35 tok/s — slowest in the suite, lowest VRAM ceiling
- Temperature=1 is intentional (GLM was trained at this; lowering did not improve tool-call reliability)

| variant | num_ctx | gen tok/s |
|---------|---------|-----------|
| `opencode_no_think` | 43520 | ~35 |
| `opencode_slow_max_ctx_thinking` | 202752 | ~10 |

VRAM cliff: between 43520 and 44032 tokens.

---

## Usage

### Build a model card
```bash
ollama create gemma4:26b_opencode_no_think -f modelfiles/Modelfile.gemma4-26b-opencode-no_think
```

### Register in opencode
Copy `opencode.example.json` to `~/.config/opencode/opencode.json` (or merge the `models` block into your existing config). The `baseURL` points to the default local ollama endpoint — change it if your instance runs elsewhere.

---

## Key findings

- **Always set `num_ctx`** — ollama defaults to 2048 which truncates opencode's system prompt and makes the model appear not to understand the harness.
- **Tool recovery scales with model size** — models below ~25B parameters tend to accept a failed tool result and stop rather than investigating further. This is a model capability issue, not fixable via model card.
- **VRAM cliff benchmarking** — test in 4K–8K increments around the suspected cliff, then narrow with 1K–2K steps to find the exact boundary. The cliff is sharp (one step from full speed to partial offload).
- **Temperature=1 is fine for GLM and Gemma** — trained at this setting. The SYSTEM instruction is the effective fix for non-empty content alongside tool calls, not temperature.
- **`/no_think` only applies to Qwen3** — GLM and Gemma handle thinking internally via their PARSER.
- **gemma4:e4b (8B) is not worth it** — tested and removed: ~48 tok/s (slower than e2b's ~78), same tool-recovery failure as e2b, no compensating advantage. Skip it; use e2b for speed or 26b for capability.
