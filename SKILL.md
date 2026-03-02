---
title: Code Ghostwriter
description: Automatically generates comprehensive documentation and comments for existing codebases using static analysis and templating.
version: "1.0.0"
author: OpenClaw Engineering
tags:
  - documentation
  - writing
  - automation
maintainer: ops@openclaw.io
dependencies:
  - openclaw-core>=2.0.0
  - tree-sitter>=0.20.0
  - jinja2>=3.0.0
---

# OpenClaw Skill: Code Ghostwriter

## Purpose

Code Ghostwriter addresses the common problem of undocumented or poorly documented codebases. It automatically injects clean, consistent, and style-compliant documentation comments and docstrings into existing source files without altering the underlying logic.

Real-world use cases:

- **Legacy code modernization**: Add documentation to old codebases that lack any comments.
- **API documentation generation**: Create JSDoc/Sphinx/Google-style docs for public APIs.
- **Team onboarding**: Improve code readability for new developers joining a project.
- **Compliance enforcement**: Meet documentation requirements for regulated industries.
- **CI/CD integration**: Automatically generate documentation as part of the build pipeline.

## Scope

Code Ghostwriter provides the following CLI commands under the `openclaw code-ghostwriter` namespace:

### `openclaw code-ghostwriter scan <path>`
Scans the specified directory or file to identify undocumented elements.

Options:
- `--language=<lang>`: Filter by language (python, javascript, typescript, go, rust, java). Default: auto-detect.
- `--exclude=<pattern>`: Exclude files matching glob pattern (e.g., `tests/*`, `*.d.ts`).
- `--min-coverage=<pct>`: Only report files with documentation coverage below threshold (0-100). Default: 0.
- `--output=<format>`: Output format (`json`, `yaml`, `text`). Default: `text`.

Example:
```bash
openclaw code-ghostwriter scan src/ --language=python --min-coverage=50 --output=json
```

### `openclaw code-ghostwriter generate [--apply]`
Generates documentation for the scanned codebase.

Options:
- `--format=<style>`: Documentation style (google, numpy, sphinx, jsdoc, java). Default: auto-detect from project.
- `--dry-run`: Show diff without modifying files.
- `--backup=<suffix>`: Create backup files with given suffix before applying. Default: `.ghostwriter.bak`.
- `--max-line-length=<cols>`: Wrap docstrings at column width. Default: 88 (PEP 257).
- `--include-tests`: Generate docs for test functions as well.

Example:
```bash
openclaw code-ghostwriter generate --format=google --dry-run
openclaw code-ghostwriter generate --apply --backup=.bak
```

### `openclaw code-ghostwriter revert`
Reverts the most recent documentation generation applied by this skill.

Options:
- `--commit`: If repository is under Git, create a revert commit automatically.
- `--files=<list>`: Revert only specific files (comma-separated).

Example:
```bash
openclaw code-ghostwriter revert --commit
```

### `openclaw code-ghostwriter config`
Manage configuration profiles.

Subcommands:
- `config set <key>=<value>`: Set a configuration option.
- `config get <key>`: Get a configuration value.
- `config list`: Show all configuration.
- `config load <profile>`: Load a predefined profile (e.g., `strict`, `fast`, `minimal`).

Global configuration stored in `~/.openclaw/code-ghostwriter/config.json`.

## Work Process

1. **Discovery**
   - The skill walks the provided directory tree recursively.
   - It identifies source files by extension (`.py`, `.js`, `.ts`, `.go`, `.rs`, `.java`).
   - Language detection uses both file extension and shebang lines.

2. **Parsing**
   - For each file, a language-specific parser (Tree-sitter) constructs an Abstract Syntax Tree (AST).
   - The AST is traversed to extract definitions: functions, classes, methods, modules.
   - Existing docstrings/comments are detected and their style is inferred.

3. **Coverage Analysis**
   - For each definition, determine if documentation is present and sufficient.
   - Coverage is computed as (documented elements / total elements) * 100.
   - A report is generated showing per-file coverage and missing items.

4. **Generation**
   - For undocumented elements, a documentation generator produces comment strings:
     - Python: Generates docstrings in chosen style (Google/Napoleon, NumPy, Sphinx, reST).
     - JavaScript/TypeScript: Generates JSDoc blocks with `@param`, `@returns`, `@throws`.
     - Go: Generates GoDoc comments preceding the declaration.
     - Rust: Generates `///` documentation comments with examples.
     - Java: Generates Javadoc style `/** ... */`.
   - The generator uses templates and, optionally, AI assistance (when connected to an LLM) to infer parameter descriptions from names and types.

5. **Validation**
   - Generated docs are checked for compliance with the selected style guide.
   - Length and completeness are verified (e.g., all parameters documented, return type described).
   - Optionally, a linter (e.g., `pydocstyle`, `eslint-plugin-jsdoc`) runs in dry-run mode to catch issues.

6. **Application**
   - In `--apply` mode, the skill inserts the generated comments into the original files.
   - Insertion happens at the correct location (immediately before function/class definition).
   - Files are rewritten atomically: write to a temp file then rename.
   - If `--backup` is set, original files are copied before modification.

7. **Post-apply checks**
   - The skill can optionally run a build/lint to ensure no syntax errors were introduced.
   - A summary report is printed showing total files changed, additions, and coverage improvement.

## Golden Rules

1. **Never alter code semantics** – The skill only adds comments; it must not reformat, rename, or move code.
2. **Preserve existing documentation** – If a function already has a docstring, do not overwrite unless `--force` flag is explicitly used.
3. **Respect project conventions** – Auto-detect style from existing docs; fall back to project config (`.docstyle`, `tsconfig.json`, `pyproject.toml`).
4. **Test file policy** – By default, skip `test_*.py`, `*.spec.js`, `__tests__` directories unless `--include-tests` is passed.
5. **Atomic writes** – Always write to a temporary file and rename to avoid partial writes on interruption.
6. **Backup before apply** – Unless disabled with `--no-backup`, keep a copy of the original file with `.ghostwriter.bak` suffix.
7. **No AI hallucination** – When using AI to generate descriptions, constrain to parameter names and types; never invent behavior not evident from code.
8. **Encodings** – Preserve original file encoding (detect BOM, use UTF-8 as default).
9. **Line endings** – Preserve original line endings (CRLF vs LF).
10. **Git awareness** – If a Git repository is detected, stage documentation-only changes separately from code changes for easier review.

## Examples

### Example 1: Basic Python docstring generation

**Before** (`src/utils.py`):
```python
def format_currency(amount, currency='USD'):
    if currency == 'USD':
        return f'${amount:.2f}'
    elif currency == 'EUR':
        return f'€{amount:.2f}'
    return str(amount)
```

**Command**:
```bash
openclaw code-ghostwriter scan src/utils.py
openclaw code-ghostwriter generate --format=google --dry-run
```

**Output (diff)**:
```diff
--- a/src/utils.py
+++ b/src/utils.py
@@ -1,6 +1,13 @@
 def format_currency(amount, currency='USD'):
+    """Formats a numeric amount as a currency string.
+
+    Args:
+        amount (float): The monetary amount to format.
+        currency (str): Currency code, e.g., 'USD', 'EUR'. Defaults to 'USD'.
+
+    Returns:
+        str: Formatted currency string with symbol.
+    """
     if currency == 'USD':
         return f'${amount:.2f}'
     elif currency == 'EUR':
```

**Apply**:
```bash
openclaw code-ghostwriter generate --apply
```

### Example 2: JavaScript/TypeScript JSDoc with type inference

**Before** (`lib/math.ts`):
```typescript
export function sum(...nums: number[]): number {
    return nums.reduce((a, b) => a + b, 0);
}
```

**Command**:
```bash
openclaw code-ghostwriter generate --language=typescript --format=jsdoc
```

**After** (`lib/math.ts`):
```typescript
/**
 * Calculates the sum of provided numbers.
 *
 * @param nums - Rest parameter of numbers to sum.
 * @returns The total sum as a number.
 */
export function sum(...nums: number[]): number {
    return nums.reduce((a, b) => a + b, 0);
}
```

### Example 3: Go documentation

**Before** (`cmd/server.go`):
```go
func startServer(port int, env string) error {
    // ...
}
```

**Command**:
```bash
openclaw code-ghostwriter generate --language=go
```

**After**:
```go
// startServer initializes and starts the HTTP server.
//
// port is the TCP port to listen on. env is the environment name (dev/staging/prod).
// Returns an error if the server fails to start.
func startServer(port int, env string) error {
    // ...
}
```

## Rollback

If documentation generation produces undesirable results, revert using:

```bash
# Revert the most recent apply operation (uses backups or Git)
openclaw code-ghostwriter revert
```

If the repository uses Git and changes were committed automatically (with `--commit`), revert via:

```bash
openclaw code-ghostwriter revert --commit
# Equivalent to: git revert HEAD
```

If you need to revert specific files manually:
```bash
# Restore from backup
cp src/module.py.ghostwriter.bak src/module.py
# Or if Git was used and you haven't committed yet:
git checkout -- src/module.py
```

The skill stores metadata of applied changes in `~/.openclaw/code-ghostwriter/last_run.json`. Ensure this file exists before reverting.

## Dependencies & Requirements

- **Python**: 3.8 or higher.
- **openclaw-core**: The core OpenClaw framework (automatically installed with skill).
- **tree-sitter**: Multi-language parsing library (automatically pulled).
- **Jinja2**: Template engine for custom docstring formats.
- **Optional AI**: Integration with LLM providers (OpenAI, Anthropic, local Ollama) for richer descriptions; set `CODE_GHOSTWRITER_LLM_API_KEY` and `CODE_GHOSTWRITER_LLM_MODEL`.
- **System tools**: `git` recommended for easy rollback; `sed`, `awk` for processing (used internally).

Installation:
```bash
openclaw skill install code-ghostwriter
```

Configuration:
```bash
openclaw code-ghostwriter config set llm.provider=ollama
openclaw code-ghostwriter config set llm.model=llama3
openclaw code-ghostwriter config set style.default=google
```

## Verification

After generating documentation, verify quality:

1. **Coverage check**:
   ```bash
   openclaw code-ghostwriter scan . --min-coverage=80
   ```
   Should show at least 80% of public functions documented.

2. **Style validation**:
   ```bash
   # For Python
   pydocstyle src/
   # For JavaScript
   npx eslint src/ --plugin jsdoc
   ```

3. **Build integrity**:
   ```bash
   # Ensure no syntax errors introduced
   python -m py_compile src/*.py
   npm run build
   ```

4. **Functional test** (if applicable):
   Run unit tests to confirm no behavioral change:
   ```bash
   pytest
   npm test
   ```

## Troubleshooting

### Issue: "Unsupported language" error
- **Cause**: File extension not recognized or parser missing.
- **Fix**: Install additional Tree-sitter grammars. List available: `tree-sitter --list-languages`. Use `--language` to force or install missing parser via npm/pip.

### Issue: Incorrect docstring style applied
- **Cause**: Auto-detection failed or project lacks configuration.
- **Fix**: Explicitly set style with `--format=google|numpy|...` or create a config file: `.docstyle` with `style = "jsdoc"`.

### Issue: "Permission denied" when writing files
- **Cause**: Output files are read-only or owned by another user.
- **Fix**: Adjust file permissions: `chmod u+w <file>`. Ensure you own the files or use `sudo` if appropriate (not recommended).

### Issue: Duplicate docstrings generated
- **Cause**: Existing docstring was present but not detected due to non-standard format.
- **Fix**: Use `--force` to overwrite existing, or improve detection by adding a custom regex pattern in config: `code-ghostwriter config set detection.patterns='["^/\*\*", "^\"\"\""]'`.

### Issue: AI-generated descriptions are inaccurate
- **Cause**: LLM hallucination or insufficient context.
- **Fix**: Disable AI assistance: `code-ghostwriter config set ai.enabled=false`. Or provide better context via `--context-file=README.md`.

### Issue: Performance too slow on large codebase
- **Fix**: Limit scanning to specific directories: `openclaw code-ghostwriter scan src/core/`. Increase parallelism with `--jobs=N` (if supported).

## Support

For issues, feature requests, or contributions, visit: https://github.com/openclaw/code-ghostwriter

Report logs with `openclaw code-ghostwriter scan --debug --output=log > debug.log`.
```