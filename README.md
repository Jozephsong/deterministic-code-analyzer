# deterministic-code-analyzer

An [opencode](https://opencode.ai) command that performs **static code analysis with maximally deterministic output** — applying every known technique to produce the same JSON on every run against the same codebase.

## Quick Start

Copy `analyze.md` into your project's `.opencode/command/` directory, then run:

```
/analyze
```

## Why Determinism Is Hard with LLMs

LLMs are stochastic by default. Even with `temperature: 0`, output can vary due to:

| Source of variance | How it manifests |
|-|-|
| Free-form prose output | Different sentence structure each run |
| Unordered file processing | Files analyzed in different sequence |
| Model version drift | `model: claude-sonnet` silently upgrades |
| Date/time in prompt | "today" resolves differently across days |
| Parallel subagents | Race conditions in result merging |
| Ambiguous instructions | LLM fills gaps differently each time |
| Locale-dependent sort | `sort` without `LC_ALL=C` orders differently per system |
| No seed parameter | Anthropic API has no `seed` — temperature=0 is best-effort, not guaranteed |
| opencode harness updates | System prompt injected by opencode changes when opencode is updated |
| Context window overflow | Truncation point varies when input exceeds context limit |
| Git state leakage | Branch name, diff, or uncommitted changes contaminate context |
| Byte-level test comparison | Semantically identical JSON fails diff due to whitespace or `1` vs `1.0` |

## How This Command Achieves Determinism

### 1. Pinned Model Version

```yaml
model: opencode/claude-sonnet-4-6-20250514
```

Uses a full, dated model ID — not an alias like `claude-sonnet` or `claude-sonnet-4-6` that may silently point to a newer model after a provider update.

### 2. Structured JSON Output

The prompt enforces a strict schema with no prose allowed before or after. There is nothing to rephrase.

```
Output ONLY the following JSON. Nothing else.
```

### 3. Locale-Pinned File Ordering

Files are discovered with `LC_ALL=C find ... | LC_ALL=C sort`. The `LC_ALL=C` prefix is critical: without it, `sort` uses the system locale and orders differently across machines (e.g., case sensitivity, special characters). `LC_ALL=C` forces byte-order sorting, which is identical everywhere. The prompt also instructs the model to read files in the exact listed order with no parallelism.

### 4. No Temporal References

The prompt contains no words like "today", "current", "now", or "recent". The analysis is stateless with respect to time.

### 5. Explicit Array Ordering Rules

Every array in the output schema has a defined sort order:

- `files[]` → by `path` alphabetically
- `functions[]` → alphabetically
- `imports[]` → alphabetically, deduplicated
- `issues[]` → by `line` asc, then `severity`, then `message`
- `languages[]` → alphabetically

### 6. No Subagents

The prompt explicitly disables subagent parallelism (`do not parallelize`), which prevents race-condition variance in result merging.

### 7. Objective-Only Issue Rules

Issues are defined as pattern matches against a fixed, closed list of rules (`todo_fixme`, `hardcoded_secret`, `long_function`, `unused_import`). There is no open-ended "find any problems" instruction that would produce different findings each run.

### 8. No Git State

The prompt explicitly prohibits reading git metadata (history, branch, diff). Git state changes frequently and is a common contamination source when agents are run mid-development.

### 9. Context Window Guard

If more than 200 files are discovered, the command stops immediately and returns a structured error instead of silently truncating. Truncation at different points in a large file list is a hidden source of variance.

### 10. Semantic JSON Comparison for Testing

`temperature: 0` with Anthropic's API is best-effort — there is no `seed` parameter, and floating-point differences across GPU inference runs can occasionally flip a token. Always compare output semantically, not by byte diff:

```bash
# Canonical comparison (not byte diff)
jq --sort-keys '.' output_run1.json > canon1.json
jq --sort-keys '.' output_run2.json > canon2.json
diff canon1.json canon2.json
```

## Output Schema

```json
{
  "analysis_version": "1.0",
  "files": [
    {
      "path": "./src/index.ts",
      "language": "typescript",
      "lines": 142,
      "functions": ["fetchUser", "parseToken", "renderApp"],
      "imports": ["express", "react", "zod"],
      "issues": [
        {
          "line": 87,
          "severity": "info",
          "rule": "todo_fixme",
          "message": "TODO: handle rate limit errors"
        }
      ]
    }
  ],
  "summary": {
    "total_files": 12,
    "total_lines": 1840,
    "languages": ["python", "typescript"],
    "issue_counts": {
      "error": 0,
      "warning": 3,
      "info": 7
    }
  }
}
```

## Supported Languages

| Extension | `language` value |
|-|-|
| `.c` | `c` |
| `.cpp`, `.h` | `cpp` |
| `.go` | `go` |
| `.java` | `java` |
| `.js`, `.jsx` | `javascript` |
| `.py` | `python` |
| `.rb` | `ruby` |
| `.rs` | `rust` |
| `.swift` | `swift` |
| `.ts`, `.tsx` | `typescript` |

## Detected Issue Rules

| Rule | Severity | Description |
|-|-|-|
| `todo_fixme` | info | Lines containing `TODO` or `FIXME` |
| `hardcoded_secret` | error | Variable containing `password`, `api_key`, `secret`, or `token` assigned a string literal |
| `long_function` | warning | Function or method body exceeding 100 lines |
| `unused_import` | warning | Import that does not appear anywhere else in the file |

## Excluded Paths

The following directories are automatically excluded from analysis:

`node_modules`, `.git`, `dist`, `build`, `.next`, `__pycache__`, `.cache`, `.turbo`, `coverage`

## Adapting the Command

**Change analyzed extensions:** Edit the `find` command in Step 1.

**Add an issue rule:** Add it to the fixed list in Step 2 with an exact, pattern-based definition. Avoid open-ended rules like "find bugs" — these are a primary source of variance.

**Change the model:** Update the `model` frontmatter field. Always use a full dated version ID. After changing the model, re-run on a reference codebase and compare output to establish a new baseline.

**Add an `$ARGUMENTS` filter:** You can append `$ARGUMENTS` to the find command to limit analysis to a subdirectory:

```
!`find ${ARGUMENTS:-.} -type f ...`
```

## GPU / Inference Server Determinism

If you are self-hosting the model (e.g., via vLLM) rather than using the Anthropic API, there are additional sources of non-determinism at the GPU and inference-server layers.

### CUDA / PyTorch Environment Variables

Set these before launching the inference server:

```bash
# Deterministic cuBLAS matrix operations (most impactful)
export CUBLAS_WORKSPACE_CONFIG=:4096:8

# Fix Python hash randomization (affects dict ordering etc.)
export PYTHONHASHSEED=0

# Deterministic NCCL all-reduce (multi-GPU only)
export NCCL_DETERMINISTIC=1
export TORCH_NCCL_USE_COMM_NONBLOCKING=0
```

In PyTorch code:

```python
import torch

torch.use_deterministic_algorithms(True)    # raises error if non-deterministic op is used
torch.backends.cudnn.deterministic = True   # force deterministic cuDNN algorithms
torch.backends.cudnn.benchmark = False      # disable auto-tuning (changes algorithm per input shape)
torch.manual_seed(42)
torch.cuda.manual_seed_all(42)
```

> `cudnn.benchmark = True` is often a default in model repos. Leaving it on means a different convolution algorithm may be selected depending on input shape, producing different outputs.

### vLLM Batch Processing Non-Determinism

vLLM uses **continuous batching**: requests that arrive at similar times are grouped into a single forward pass. This is a real source of non-determinism even at temperature=0.

**Why batching changes output:**

```
Request A alone:      [tok_A1, tok_A2, PAD,   PAD,   PAD  ]
Request A with B:     [tok_A1, tok_A2, tok_B1, tok_B2, tok_B3]
```

The same request A produces different attention computation patterns depending on what it is batched with. Flash Attention is not bit-exact across different padding configurations, so the output token distribution shifts slightly — enough to change the selected token under greedy decoding.

**vLLM launch flags:**

```bash
vllm serve <model> \
  --seed 42 \
  --enforce-eager \                  # disable CUDA graph capture
  --disable-custom-all-reduce        # use standard NCCL (more deterministic)
```

**Fixing batch composition (most effective approach):**

```python
from vllm import LLM, SamplingParams

llm = LLM(
    model="...",
    seed=42,
    enforce_eager=True,
    max_num_seqs=1,       # process one request at a time — eliminates batch variance
)
```

`max_num_seqs=1` eliminates batch-composition variance entirely but reduces throughput. If throughput matters, the practical compromise is to **sort inputs by length** before submission so that batch composition is always identical for the same input set:

```python
prompts_sorted = sorted(prompts, key=len)
outputs = llm.generate(prompts_sorted, SamplingParams(temperature=0, seed=42))
```

**Flash Attention vs xFormers:**

Flash Attention has known non-determinism under certain padding configurations. Switching to xFormers can help:

```bash
export VLLM_ATTENTION_BACKEND=XFORMERS
```

### Multi-GPU (Tensor Parallelism)

All-reduce operations sum floating-point values across GPUs in a hardware-dependent order. The result differs from single-GPU inference and differs across different numbers of GPUs. **Keep the number of GPUs fixed** across runs to maintain consistent output.

### Summary Table

| Layer | Setting | Impact |
|-|-|-|
| CUDA | `CUBLAS_WORKSPACE_CONFIG=:4096:8` | High |
| PyTorch | `cudnn.deterministic=True`, `benchmark=False` | High |
| vLLM batching | Sort inputs by length, or `max_num_seqs=1` | High |
| vLLM launch | `--seed 42 --enforce-eager` | Medium |
| Multi-GPU | Fix GPU count, `NCCL_DETERMINISTIC=1` | High |
| Python | `PYTHONHASHSEED=0` | Low |

---

## Exhaustive Non-Determinism Reference

This section documents every known source of non-determinism in LLM-based agents, beyond what the `analyze.md` command directly addresses. Use it as a checklist when debugging reproducibility issues in any LLM pipeline.

---

### Floating-Point Precision

**Root cause:** Floating-point arithmetic is non-associative — `(a+b)+c ≠ a+(b+c)`. GPU parallel reductions sum values in different orders depending on batch size, parallelism degree, and kernel implementation. This is the single deepest source of LLM non-determinism.

**Precision format impact (from worst to best):**

| Format | Mantissa bits | Divergence rate | Notes |
|-|-|-|-|
| BF16 | 7 | ~90% of samples diverge | Default on most modern inference stacks |
| FP16 | 10 | Moderate | Better than BF16, worse than FP32 |
| FP32 | 23 | ~2.2% diverge | Near-perfect reproducibility |

**TF32 (hidden default on Ampere/Hopper GPUs):** NVIDIA GPUs silently use TF32 (19-bit) for matrix multiplies unless disabled. This introduces rounding invisible to the caller.

```python
# Disable TF32 to get true FP32 behavior
torch.backends.cuda.matmul.allow_tf32 = False
torch.backends.cudnn.allow_tf32 = False
```

**LayerCast (best of both worlds):** Store weights in BF16 (34% less memory than FP32), upcast to FP32 just before each computation. Achieves near-FP32 determinism at BF16 memory cost.

```python
# In vLLM
llm = LLM(model="...", dtype="bfloat16")
# Or force full FP32:
llm = LLM(model="...", dtype="float32")
```

---

### Batch-Dependent Kernel Behavior

Even with fixed inputs, GPU kernels select different reduction strategies based on batch size. This is distinct from batch *composition* — it affects a **single request** processed under different server load.

**Specific mechanisms:**
- **Tensor-core instruction switching:** Below certain batch size thresholds, kernels switch from large tensor-core instructions to smaller ones or scalar ops, changing the internal accumulation order.
- **Split-KV strategies in attention:** Typical Flash Attention implementations dynamically choose the number of KV splits based on available parallelism. Different split counts → different reduction order → different logit values.
- **KV cache boundary:** Processing 999 cached tokens + 1 new token uses a different kernel path than processing 1000 new tokens. The boundary between cached and current KV creates a different floating-point reduction sequence.
- **MatMul batch extraction:** A matrix multiplication extracted from a batch of size N produces different results than the same multiplication run alone, by up to thousands of units in raw logit space.

**Mitigation:** Use batch-invariant kernel implementations (fixed data-parallel strategy regardless of batch size) or `max_num_seqs=1`.

---

### vLLM Prefix Caching (KV Cache Hash Collisions)

vLLM's Automatic Prefix Caching hashes token sequences to find reusable KV cache blocks. The hash algorithm matters for reproducibility:

| Algorithm | Serializer | Cross-run stable? | Notes |
|-|-|-|-|
| `sha256` (default) | Python pickle | No | Hash may differ across Python/vLLM versions |
| `sha256_cbor` | cbor2 | Yes | Reproducible and cross-language compatible |
| `xxhash` | pickle | No | Fast but non-cryptographic, pickle-dependent |

```python
# vLLM: use stable hash algorithm
llm = LLM(model="...", prefix_caching_hash_algo="sha256_cbor")
```

Also, when a prefix cache *hit* occurs, the KV layout differs from a fresh computation — the attention kernel processes cached and current KV separately, changing the reduction boundary and potentially the output token.

---

### Tokenizer Layer

**Unicode normalization:** The same visible string can be encoded differently in Unicode (NFC, NFD, NFKC, NFKD). "é" as a single codepoint vs. "e" + combining accent produces different tokens. Copy-pasted text from different applications often differs invisibly.

```python
import unicodedata
prompt = unicodedata.normalize("NFC", prompt)  # always normalize before tokenizing
```

**Invisible characters:** Zero-width spaces (U+200B), BOM (U+FEFF), soft hyphens (U+00AD), and directional marks are tokenized differently across models. A prompt copy-pasted from a web page may contain these invisibly.

**BPE boundary sensitivity:** BPE tokenizers are sensitive to whitespace at chunk boundaries. If a file is read with trailing whitespace on one run and without on another, the tokenization of subsequent content can shift. This matters when concatenating file contents into a prompt.

**Tokenizer version drift:** The tokenizer is a separate artifact from the model weights. Updating `transformers` or `tokenizers` libraries can silently change how edge-case strings are tokenized even for the same model.

```bash
# Pin tokenizer library version
pip install transformers==4.44.0 tokenizers==0.19.1
```

---

### Speculative Decoding

Speculative decoding uses a small draft model to propose tokens, which the target model then accepts or rejects. This introduces non-determinism beyond temperature:

- **Draft model acceptance rate varies with batch size:** Even at temperature=0, changes in batch size alter logprobs in the draft model, changing which tokens are accepted.
- **Draft model version:** Different draft models (or draft model updates) produce different proposal sequences, leading to different acceptance patterns and different final outputs even when the target model is identical.
- **Token budget interaction:** If the number of speculative tokens per step varies (e.g., due to a dynamic draft), the execution path of the target model changes.

**Mitigation:** Disable speculative decoding for determinism-critical workloads, or fix both the draft model version and the number of speculative tokens.

```python
# Disable speculative decoding in vLLM
llm = LLM(model="...", speculative_model=None)
```

---

### Mixture of Experts (MoE) Routing

MoE models (Mixtral, DeepSeek, Qwen-MoE, etc.) route each token to a subset of expert networks. The routing is a non-trivial source of non-determinism:

- **Tied routing scores:** When two experts have nearly equal scores, floating-point tie-breaking is hardware-dependent. Different GPU architectures (A100 vs H100) resolve ties differently.
- **Expert parallelism dispatch order:** When experts are distributed across GPUs, the all-to-all dispatch operation is non-deterministic in ordering under high load.
- **Capacity factor effects:** If an expert is at capacity (token dropping in training-style routing), which tokens are dropped depends on the order tokens arrive at the expert.

**Mitigation:** For MoE models, additionally set:
```bash
export CUDA_LAUNCH_BLOCKING=1  # serialize CUDA ops (large throughput penalty)
```
Or accept that MoE routing adds a baseline of non-determinism that is difficult to fully eliminate without custom kernel changes.

---

### Sampling Parameter Interactions

Beyond temperature and top_p, these parameters affect token selection:

| Parameter | Effect on determinism |
|-|-|
| `repetition_penalty` | Modifies logits based on prior output history; same prompt with different history → different output |
| `presence_penalty` | Penalizes tokens seen in context; context-dependent |
| `frequency_penalty` | Scales penalty by token frequency in context |
| `logit_bias` | Fixed additive bias — deterministic if constant, non-deterministic if computed dynamically |
| `min_p` | Minimum probability threshold — can change the effective candidate set |
| `top_k` | Limits candidate set; at boundaries, floating-point sort order is hardware-dependent |

**Rule:** Any parameter computed from dynamic state (context length, token history) must be computed identically across runs.

---

### Prompt Caching (Anthropic Server-Side)

Anthropic's prompt caching stores KV states for repeated prompt prefixes. The cache itself does **not** change output quality — a cache hit produces the same result as a cache miss for the completion tokens. However, the cache is a non-determinism source in a different way:

- **Dynamic content invalidates the cache**, forcing a cold computation that may follow a slightly different GPU execution path than the cached path.
- **Python `dict`/`set` ordering in tool lists:** Before Python 3.7, dict iteration order is undefined. If tool definitions are built from a set or dict and serialized into the prompt, two Python processes may produce different orderings → different prompt hash → different cache entry → possible output difference.
- **Cache TTL (5 minutes):** A prompt cached at minute 0 and re-run at minute 6 takes a full (non-cached) path. If the non-cached path is on a different GPU instance, floating-point differences apply.

```python
# Always build tool lists from sorted, deterministic structures
tools = sorted(tool_definitions, key=lambda t: t["name"])
```

---

### torch.compile and CUDA Graphs

`torch.compile` and CUDA graph capture introduce their own non-determinism:

- **Graph selection by batch size:** vLLM captures CUDA graphs for a discrete set of batch sizes and selects the smallest graph that fits. A batch of 7 runs the graph captured for batch 8, with padded inputs. Different padding → different result.
- **Compilation cache invalidation:** If the compiled artifact cache is stale or corrupted, a recompilation may produce a numerically different graph (different operator fusion decisions).
- **Eager vs compiled paths:** `--enforce-eager` disables CUDA graph capture and runs every operation in eager mode. This is more deterministic but slower.

```bash
# Most deterministic vLLM configuration
vllm serve <model> \
  --enforce-eager \
  --seed 42 \
  --disable-custom-all-reduce \
  --dtype float32
```

---

### Quantization

**Post-training quantization is not bit-exact:**

- **GPTQ** quantizes in batches using MSE minimization. The quantized weights are fixed after calibration, but the calibration dataset itself affects weight values — a different calibration set produces a different quantized model.
- **AWQ** identifies ~1% of salient weight channels using activation statistics. Group-wise quantization can disrupt instruction alignment even when weight fidelity is preserved.
- **INT4/INT8 rounding:** Quantized models accumulate rounding error per layer. These errors are deterministic for the same weights but produce different results from the FP16 baseline.

**Mitigation:** Treat each quantized model as a distinct model version. Version-control quantization parameters, group size, and calibration dataset alongside the weights. Never compare outputs of different quantization configurations directly.

---

### Framework and Library Version Pinning

Each layer of the software stack can change numerical behavior across versions:

```bash
# Pin everything that touches matrix math
pip install \
  torch==2.4.0 \
  vllm==0.6.1 \
  transformers==4.44.0 \
  tokenizers==0.19.1 \
  flash-attn==2.6.3
```

Additionally pin at the system level:
```bash
# Check and document CUDA and cuDNN versions
nvcc --version
python -c "import torch; print(torch.backends.cudnn.version())"
```

New PyTorch minor versions routinely change operator implementations. A `pip install --upgrade` mid-project is a common unnoticed source of output drift.

---

### OS and System Layer

These are low-probability but have been observed in production:

- **NUMA topology:** On multi-socket servers, memory bandwidth differences between NUMA nodes can cause timing-dependent kernel selection that affects floating-point reduction order.
- **CPU affinity:** If the inference process's CPU affinity changes between runs (e.g., due to container scheduling), PyTorch's CPU thread pool size may change, affecting data-loading order.
- **Filesystem read ordering:** `os.listdir()` and `glob.glob()` return files in filesystem-dependent order (inode order on ext4, creation order on some systems). Always `sorted()` any directory listing used to construct prompts.
- **System clock resolution:** Any code path that reads `time.time()` or `datetime.now()` and includes it in a prompt or file name used as input introduces per-run variance.

---

### Agent Loop Layer

- **Max iteration limits:** If the agent hits a loop cap, it terminates with partial output. A loop cap reached on run N but not run N+1 (because run N+1 was slightly faster) produces structurally different output.
- **Token budget early termination:** A cost or token budget that causes `max_tokens` to be hit mid-JSON produces unparseable output. The truncation point varies with the exact token count of intermediate reasoning.
- **Error recovery paths:** If a tool call fails and the agent retries, the retry's output is appended to context. The same prompt with and without a prior retry attempt in context produces different output.
- **Timeout-triggered fallbacks:** Request timeout causing a fallback to a smaller/different model is a total model switch, not just a small numerical difference.

---

### Complete Checklist

| Layer | Source | Fix |
|-|-|-|
| Floating-point | BF16 non-associativity | Use FP32 or LayerCast |
| Floating-point | TF32 default on Ampere | `allow_tf32 = False` |
| Kernel | Batch-dependent reduction strategy | Batch-invariant kernels or `max_num_seqs=1` |
| Kernel | Split-KV attention variance | `--enforce-eager` |
| Kernel | Tensor-core instruction switching | Batch-invariant build |
| vLLM | Prefix cache hash instability | `sha256_cbor` algorithm |
| vLLM | Cache hit/miss path divergence | `--no-enable-prefix-caching` for strict determinism |
| vLLM | CUDA graph batch size selection | `--enforce-eager` |
| Tokenizer | Unicode normalization | `unicodedata.normalize("NFC", prompt)` |
| Tokenizer | Invisible Unicode characters | Strip/audit prompts |
| Tokenizer | BPE whitespace boundary | Normalize whitespace in file content |
| Tokenizer | Library version drift | Pin `transformers` and `tokenizers` |
| Speculative decoding | Draft acceptance variance | Disable or fix draft model + token count |
| MoE | Expert routing tie-breaking | Accept baseline variance or `CUDA_LAUNCH_BLOCKING=1` |
| Sampling | Dynamic logit modifiers | Ensure all penalty params are constant |
| Prompt caching | Cache miss path divergence | Keep prompt prefix static |
| Prompt caching | dict/set ordering in tool lists | `sorted()` before serializing |
| Quantization | Calibration-dependent weights | Version-control quant params + calibration data |
| Framework | Version drift | Pin all packages including CUDA/cuDNN |
| OS | Filesystem listing order | Always `sorted()` directory listings |
| OS | NUMA/CPU affinity | Fix container CPU pinning |
| Agent loop | Max iteration / token budget | Set generous limits; monitor for early exit |
| Agent loop | Retry context pollution | Make retries stateless or avoid retries |
| Multi-GPU | all-reduce ordering | Fix GPU count; `NCCL_DETERMINISTIC=1` |

---

## Caveats

- **temperature=0 is not a hard guarantee.** Anthropic's API has no `seed` parameter. Floating-point differences across GPU inference instances can occasionally produce different tokens when two candidates have near-equal probability. Compare outputs semantically (see Section 10 above), not byte-for-byte.
- **opencode harness updates reset the baseline.** opencode injects its own system prompt before your command runs. If opencode is updated, its system prompt may change, which can shift model behavior. Re-verify determinism after any opencode update.
- **Rotating to a new model version resets the baseline.** Re-run on a reference codebase and establish a new expected output after any model change.
- **The `unused_import` rule may have false positives** on re-exported or dynamically accessed identifiers, since it relies on the LLM's text-search reasoning rather than a real AST parser.
