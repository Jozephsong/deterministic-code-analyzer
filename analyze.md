---
description: "Deterministic static code analysis of the current directory"
model: opencode/claude-sonnet-4-6-20250514
---

Perform a deterministic static code analysis of the current directory. Follow every instruction exactly as written. Produce the same output every time this command is run on the same codebase.

## Determinism Rules (follow strictly)

1. **No temporal references** — do not mention "today", "current date", timestamps, or any time-dependent information
2. **Alphabetical ordering** — process all files, lists, and arrays in alphabetical/ascending order unless otherwise specified
3. **Exact output format** — output ONLY the JSON block defined in Step 3. No prose, headers, or explanation before or after.
4. **Omit uncertainty** — if a finding is ambiguous or you are not confident, omit it rather than guess
5. **Fixed file filter** — only analyze source files matching the extensions listed in Step 1
6. **No subagents** — process files sequentially in the order listed; do not parallelize

## Step 1: Discover Source Files

Run this exact command and use its output as your file list:

!`find . -type f \( -name "*.ts" -o -name "*.tsx" -o -name "*.js" -o -name "*.jsx" -o -name "*.py" -o -name "*.go" -o -name "*.rs" -o -name "*.java" -o -name "*.rb" -o -name "*.swift" -o -name "*.c" -o -name "*.cpp" -o -name "*.h" \) | grep -v -E "(node_modules|\.git|dist|build|\.next|__pycache__|\.cache|\.turbo|coverage)" | sort`

Read each file in the exact order listed above. Do not read files in any other order.

## Step 2: Analyze Each File (in listed order)

For each file, extract the following. Apply each rule identically to every file:

- **language**: detected from file extension only (use these exact strings: "c", "cpp", "go", "java", "javascript", "python", "ruby", "rust", "swift", "typescript")
- **lines**: total line count (including blank lines and comments)
- **functions**: names of all top-level functions, methods, and class definitions found in the file, sorted alphabetically. Use the exact name as it appears in the source.
- **imports**: module/package names only from import/require/use statements, sorted alphabetically, deduplicated. Strip quotes and path prefixes — use only the top-level package name (e.g. `react` not `react/hooks`, `fmt` not `fmt/errors`).
- **issues**: only objective, verifiable issues from this fixed list:
  - Lines containing `TODO` or `FIXME` (case-sensitive) → severity `"info"`, rule `"todo_fixme"`
  - Hardcoded credential patterns: variable names containing `password`, `api_key`, `secret`, `token` assigned a string literal → severity `"error"`, rule `"hardcoded_secret"`
  - Functions/methods whose body spans more than 100 lines → severity `"warning"`, rule `"long_function"`
  - Unused imports: imports that do not appear anywhere else in the file → severity `"warning"`, rule `"unused_import"`

## Step 3: Output

Output ONLY the following JSON. Nothing else — no markdown fences, no explanation, no trailing newline commentary.

```json
{
  "analysis_version": "1.0",
  "files": [
    {
      "path": "<relative path starting with ./>",
      "language": "<exact language string from Step 2>",
      "lines": 0,
      "functions": ["<sorted alphabetically>"],
      "imports": ["<sorted alphabetically, deduplicated>"],
      "issues": [
        {
          "line": 0,
          "severity": "<error|warning|info>",
          "rule": "<todo_fixme|hardcoded_secret|long_function|unused_import>",
          "message": "<concise factual description, max 120 chars>"
        }
      ]
    }
  ],
  "summary": {
    "total_files": 0,
    "total_lines": 0,
    "languages": ["<sorted alphabetically>"],
    "issue_counts": {
      "error": 0,
      "warning": 0,
      "info": 0
    }
  }
}
```

**Strict ordering rules:**
- `files` array: sorted by `path` value alphabetically (ascending)
- `functions` array: sorted alphabetically (ascending)
- `imports` array: sorted alphabetically (ascending), duplicates removed
- `issues` array: sorted by `line` ascending; ties broken by `severity` alphabetically; further ties broken by `message` alphabetically
- `languages` array: sorted alphabetically (ascending)
- `issue_counts`: always include all three keys (`error`, `warning`, `info`) even if the count is 0
- Empty arrays must be written as `[]`, not omitted
