---
name: dead-code-hunter
description: >
  Use this skill whenever the user wants to find unused, unreferenced, or dead code in their
  codebase. Trigger on phrases like: "find dead code", "what code is unused", "clean up unused
  exports", "find unreferenced files", "what functions are never called", "hunt dead code",
  "find unused functions/variables/exports", "what can I safely delete", "codebase cleanup".
  Also trigger when the user asks about code that "might not be used anymore" or wants to
  reduce codebase size. Use this aggressively — if the user mentions unused or unreferenced
  code in any form, this skill should run.
---

# Dead Code Hunter

This skill finds code that is defined but never actually used: exported functions nobody imports,
private helpers that are never called, variables that are never read, and entire files nothing
depends on.

The goal is a precise, actionable report — not a list of false positives. Dead code analysis
done naively produces noise that erodes trust. The skill should flag things with high confidence
and clearly explain *why* each item appears dead, so the user can verify quickly.

## Step 1: Understand the codebase

Before searching, get oriented:

```bash
# Detect language(s) and project structure
ls *.json *.toml *.cfg pyproject.toml go.mod Cargo.toml 2>/dev/null | head -10
find . -name "*.ts" -o -name "*.js" -o -name "*.py" -o -name "*.go" | grep -v node_modules | grep -v ".git" | wc -l

# Find source root (exclude tests, node_modules, dist, build, .git)
ls src/ app/ lib/ pkg/ 2>/dev/null
```

Identify:
- **Primary language** — TypeScript/JS, Python, Go, etc.
- **Source root** — where application code lives (usually `src/`, `app/`, `lib/`)
- **Entry points** — files that are roots of the dependency graph: `index.ts`, `main.ts`, `app.py`, CLI scripts, route files. Code only referenced from entry points is not dead.
- **Test files** — files matching `*.test.*`, `*.spec.*`, `*_test.*`, `tests/`. A symbol used only in tests is a special case — flag it separately as "test-only", not dead.

## Step 2: Search for dead code by category

Work through each category. Use `grep` and `find` heavily — this is pattern matching work, not deep static analysis. Be systematic.

### A. Exported symbols never imported elsewhere

**TypeScript / JavaScript:**
```bash
# Find all named exports
grep -rn "^export \(function\|class\|const\|let\|var\|type\|interface\|enum\)" src/ --include="*.ts" --include="*.js" | grep -v "node_modules" | grep -v ".d.ts"

# For each exported name, check if anything imports it
# Example: checking if 'formatDate' is imported anywhere
grep -rn "formatDate" src/ --include="*.ts" --include="*.js" | grep -v "^[^:]*:export"
```

Watch out for **barrel files** (`index.ts` that re-exports everything) — a symbol exported from a barrel but never imported from the barrel *or* directly is dead. A symbol that appears in a barrel re-export but is never imported by any consumer is still dead.

**Python:**
```bash
# Find all top-level function/class definitions
grep -rn "^def \|^class " src/ --include="*.py"

# Check if used elsewhere
grep -rn "function_name" . --include="*.py" | grep -v "^[^:]*:def \|^[^:]*:class "
```

### B. Unexported (private) functions never called within their own file

These are functions defined without `export` in TS/JS, or prefixed with `_` in Python, or lowercase in Go — intended to be internal, but never actually invoked.

```bash
# TS/JS: functions without export keyword
grep -n "^function \|^const .* = (" src/utils.ts | grep -v "^export"

# Then for each, check call sites within the same file
grep -n "functionName(" src/utils.ts
```

For a private function, a single call site within the same file is enough to mark it as alive.

### C. Variables and constants defined but never read

Focus on module-level constants (not local variables inside functions — those produce too much noise). A module-level `const API_TIMEOUT = 5000` that's never referenced is a genuine cleanup opportunity.

```bash
grep -n "^const \|^let \|^var " src/config.ts | grep -v export
# Then check each name for references
grep -rn "API_TIMEOUT" src/
```

### D. Entire files with no imports pointing to them

```bash
# Get all source files
find src/ -name "*.ts" ! -name "*.d.ts" ! -name "*.test.ts" | sort

# For each file, check if any other file imports it
# Example: checking if src/helpers/legacy.ts is imported anywhere
grep -rn "from.*helpers/legacy\|require.*helpers/legacy" src/
```

Entry point files (`index.ts`, `main.ts`, route files) will legitimately have no importers — exclude them from this check.

## Step 3: Filter false positives

Before reporting, apply these sanity checks to avoid crying wolf:

**Dynamic access** — If a symbol is accessed via string lookup (`obj[key]`, `require(variable)`, `getattr(obj, name)`), grep won't find it. If the codebase has patterns like this, note it as a caveat.

**Re-exports** — A symbol exported from `index.ts` that is also imported by consumers *through* the index is alive. Make sure you're checking for usage across all import styles (`import { X } from '../'`, `import { X } from '../index'`).

**External consumers** — If this is a library, exported symbols may be used by consumers outside the repo. Flag public API exports as "possibly used externally" rather than dead.

**Decorators and framework magic** — In frameworks like NestJS, Angular, or FastAPI, decorated functions/classes may be referenced by the framework at runtime without a direct import. If you see `@Injectable()`, `@Controller()`, `@app.route()` etc., be conservative.

**Config-only references** — Some symbols are referenced in config files, build scripts, or Docker files that grep on source won't catch. Mention this caveat for anything that looks infrastructure-adjacent.

## Step 4: Print the report

Group by category, sorted by file. Be concise — one line per item with enough context to verify.

```
Dead Code Report — src/ (47 files scanned)

━━ EXPORTED BUT NEVER IMPORTED ━━━━━━━━━━━━━━━━━━━━━━━━━━━
  src/utils/date.ts:12      formatLegacyDate()      — no import found in 47 files
  src/utils/date.ts:28      LEGACY_DATE_FORMAT       — no import found in 47 files
  src/api/helpers.ts:5      buildQueryString()      — no import found in 47 files

━━ PRIVATE, NEVER CALLED ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  src/services/auth.ts:84   _normalizeEmail()       — defined, 0 call sites in file
  src/utils/string.ts:201   truncateMiddle()        — defined, 0 call sites in file

━━ UNUSED VARIABLES / CONSTANTS ━━━━━━━━━━━━━━━━━━━━━━━━━━
  src/config.ts:3           MAX_RETRY_LEGACY = 3   — never referenced

━━ ORPHANED FILES ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  src/utils/deprecated.ts                           — nothing imports this file

━━ TEST-ONLY (not dead, but worth knowing) ━━━━━━━━━━━━━━━
  src/utils/testHelpers.ts  seedDatabase()          — only referenced in *.test.ts files

━━ SKIPPED (low confidence) ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  src/api/routes.ts         — uses dynamic require(), skipped
  src/plugins/             — framework decorators detected, skipped

Found 8 dead symbols across 4 files, 1 orphaned file.
⚠ This is a grep-based analysis — verify before deleting, especially re-exports and decorated classes.
```

## Tips for accuracy

- Always check multiple import styles: `import { X }`, `import X`, `import * as X`, `require('...')`, and dynamic patterns
- In TypeScript, also check `export default` separately — it's imported differently (`import X from '...'`, no braces)
- For Python, check `from module import name` AND `module.name` usage patterns
- When in doubt, report with lower confidence rather than missing real dead code — the user can verify
- A function that only calls itself (mutual recursion with no external caller) is still dead
