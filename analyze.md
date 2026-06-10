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
7. **No git state** — do not read git history, branch names, diff output, or any git metadata
8. **Context limit guard** — if the file list from Step 1 contains more than 200 files, stop and output only this JSON: `{"error": "too_many_files", "count": <actual count>, "limit": 200}`
9. **Unicode normalization** — treat all file content as NFC-normalized Unicode. If you encounter sequences that appear identical but may differ in encoding (e.g., accented characters), treat them as equivalent. Ignore invisible characters: zero-width space (U+200B), BOM (U+FEFF), soft hyphen (U+00AD), and directional marks.
10. **Whitespace normalization** — treat trailing whitespace on any line as absent. Treat tab-indented and space-indented files identically for the purpose of line counting and import detection.
11. **No retries** — if reading a file fails, record the file with `"lines": 0`, `"functions": []`, `"imports": []`, `"issues": [{"line": 0, "severity": "error", "rule": "read_error", "message": "file could not be read"}]`. Do not attempt to read the file again.
12. **File size guard** — if a file exceeds 1000 lines, record only its `path`, `language`, and `lines`. Set `functions`, `imports`, and `issues` to `[]`. Do not read beyond line 1000.
13. **No dynamic behavior** — do not adjust analysis depth, detail level, or output structure based on file count, project size, or any runtime observation. Apply every rule identically regardless of scale.

## Step 1: Discover Source Files

Run this exact command and use its output as your file list:

!`LC_ALL=C find . -type f \( -name "*.ts" -o -name "*.tsx" -o -name "*.js" -o -name "*.jsx" -o -name "*.py" -o -name "*.go" -o -name "*.rs" -o -name "*.java" -o -name "*.rb" -o -name "*.swift" -o -name "*.c" -o -name "*.cpp" -o -name "*.h" \) | grep -v -E "(node_modules|\.git|dist|build|\.next|__pycache__|\.cache|\.turbo|coverage)" | LC_ALL=C sort`

- `LC_ALL=C` is mandatory on both `find` and `sort` — without it, sort order is locale-dependent and varies across machines.
- Read each file in the exact order listed above. Do not read files in any other order.
- Apply Rule 8 (200-file limit) and Rule 12 (1000-line limit per file) before proceeding.

## Step 2: Analyze Each File (in listed order)

For each file, extract the following. Apply each rule identically to every file:

- **language**: detected from file extension only (use these exact strings: "c", "cpp", "go", "java", "javascript", "python", "ruby", "rust", "swift", "typescript")
- **lines**: total line count including blank lines and comments. Count is based on newline characters — a file with no trailing newline counts its last line.
- **functions**: names of all top-level functions, methods, and class definitions found in the file, sorted alphabetically. Use the exact name as it appears in the source. Do not infer names from comments or documentation.
- **imports**: module/package names only from import/require/use statements, sorted alphabetically, deduplicated. Strip quotes and path prefixes — use only the top-level package name (e.g. `react` not `react/hooks`, `fmt` not `fmt/errors`). Apply whitespace normalization (Rule 10) before extracting names.
- **issues**: only objective, verifiable issues from this fixed list. Do not report anything outside this list:
  - Lines containing the exact string `TODO` or `FIXME` (case-sensitive, whole string) → severity `"info"`, rule `"todo_fixme"`
  - Hardcoded credential patterns: variable names containing `password`, `api_key`, `secret`, or `token` (case-insensitive substring match) assigned a non-empty string literal on the same line → severity `"error"`, rule `"hardcoded_secret"`
  - Functions/methods whose body spans more than 100 lines (count from opening brace/colon to closing brace/dedent) → severity `"warning"`, rule `"long_function"`
  - Unused imports: imports whose top-level package name does not appear as a substring anywhere else in the file after the import block → severity `"warning"`, rule `"unused_import"`

## Step 3: Output

Output ONLY the following JSON. Nothing else — no markdown fences, no explanation, no trailing newline commentary.

```json
{
  "analysis_version": "1.1",
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
          "rule": "<todo_fixme|hardcoded_secret|long_function|unused_import|read_error>",
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

**Strict ordering rules (non-negotiable):**
- `files` array: sorted by `path` value, `LC_ALL=C` byte order (ascending)
- `functions` array: sorted alphabetically (ascending)
- `imports` array: sorted alphabetically (ascending), duplicates removed
- `issues` array: sorted by `line` ascending; ties broken by `severity` alphabetically (`error` < `info` < `warning`); further ties broken by `message` alphabetically
- `languages` array: sorted alphabetically (ascending)
- `issue_counts`: always include all three keys (`error`, `warning`, `info`) even if the count is 0
- Empty arrays must be written as `[]`, not omitted
- Numbers must be written as integers (e.g. `42`, not `42.0`)
