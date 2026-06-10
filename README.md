# deterministic-code-analyzer

An [opencode](https://opencode.ai) command that performs **static code analysis with deterministic output** — running the same command on the same codebase always produces byte-identical JSON.

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

### 3. Deterministic File Ordering

Files are discovered with `find ... | sort`, which produces stable alphabetical order on any POSIX system. The prompt instructs the model to read files in the exact listed order with no parallelism.

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

## Caveats

- `temperature: 0` reduces but does not eliminate variance. Long outputs or ambiguous patterns may still produce minor differences between runs.
- Rotating to a new model version resets the baseline — re-verify determinism after any model change.
- The `unused_import` rule relies on the LLM's text-search reasoning and may have false positives on re-exported or dynamically accessed identifiers.
