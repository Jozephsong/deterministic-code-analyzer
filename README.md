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

## Caveats

- **temperature=0 is not a hard guarantee.** Anthropic's API has no `seed` parameter. Floating-point differences across GPU inference instances can occasionally produce different tokens when two candidates have near-equal probability. Compare outputs semantically (see Section 10 above), not byte-for-byte.
- **opencode harness updates reset the baseline.** opencode injects its own system prompt before your command runs. If opencode is updated, its system prompt may change, which can shift model behavior. Re-verify determinism after any opencode update.
- **Rotating to a new model version resets the baseline.** Re-run on a reference codebase and establish a new expected output after any model change.
- **The `unused_import` rule may have false positives** on re-exported or dynamically accessed identifiers, since it relies on the LLM's text-search reasoning rather than a real AST parser.
