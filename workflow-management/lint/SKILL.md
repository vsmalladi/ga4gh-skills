---
name: wdl-lint
description: >
  Validate, lint, and fix WDL files using womtool, miniwdl check, and/or
  sprocket. Use this skill whenever the user wants to check WDL syntax,
  understand or fix lint warnings, suppress specific rules, interpret tool
  error output, set up CI validation, or run any of: womtool validate,
  miniwdl check, sprocket check, sprocket lint, or sprocket format.
  Also trigger when the user pastes tool output containing WDL errors or
  warnings and asks for help understanding or fixing them.
---

# WDL Lint & Validation

Three tools cover the WDL validation/lint space. Each has a different
focus, WDL version range, and strictness model. Use all three for maximum
coverage — they catch different things.

| Tool | Language | WDL versions | Strength |
|------|----------|-------------|----------|
| **womtool** | Java (Broad/Cromwell) | draft-2 – 1.2 | Cromwell-specific semantic validation; inputs template generation |
| **miniwdl check** | Python (CZI) | draft-2 – 1.2 | Best ShellCheck integration; richest warning names; fast feedback |
| **sprocket** | Rust (St. Jude) | 1.0 – 1.3 | Strictest linter; full 1.2/1.3 support; LSP; formatter; CI-ready |

> For WDL syntax help see the root `SKILL.md` and `references/wdl-spec-highlights.md`.
> For linting and fixing WDL, see `lint/SKILL.md`.

---

## Installation

```bash
# womtool — download the jar matching your Cromwell version
wget https://github.com/broadinstitute/cromwell/releases/latest/download/womtool-<version>.jar
alias womtool='java -jar /path/to/womtool.jar'

# miniwdl — Python 3.6+, Docker required for run (not check)
pip install miniwdl
# with ShellCheck integration (recommended)
brew install shellcheck       # macOS
sudo apt install shellcheck   # Debian/Ubuntu

# sprocket — single binary, no JVM/Python needed
cargo install sprocket        # via Rust/cargo
# or download prebuilt from GitHub releases:
# https://github.com/stjude-rust-labs/sprocket/releases
```

---

## Quick Reference — Common Commands

```bash
# ── WOMTOOL ─────────────────────────────────────────────────────────
womtool validate workflow.wdl                     # syntax + type check
womtool validate --list-dependencies workflow.wdl # also print imports
womtool inputs workflow.wdl                       # generate inputs JSON template
womtool inputs workflow.wdl > inputs.json
womtool parse workflow.wdl                        # grammar-only AST (no semantic check)
womtool graph workflow.wdl                        # DOT-format dependency graph
womtool highlight workflow.wdl html               # colorized HTML output
womtool highlight workflow.wdl console            # colorized terminal output

# ── MINIWDL ─────────────────────────────────────────────────────────
miniwdl check workflow.wdl                        # parse + type + lint (exit 0 on warn)
miniwdl check --strict workflow.wdl               # lint warnings → non-zero exit
miniwdl check -p imports/ workflow.wdl            # add import search path
miniwdl check --suppress MissingVersion,StringCoercion workflow.wdl
miniwdl check --no-quant-check workflow.wdl       # skip optional→String coercion warnings
miniwdl check workflow.wdl task.wdl               # check multiple files at once

# ── SPROCKET ────────────────────────────────────────────────────────
sprocket check workflow.wdl                       # analysis only (errors/warnings, no extra lint)
sprocket lint workflow.wdl                        # check + full lint rules (shortcut for check --lint)
sprocket lint .                                   # lint entire directory (defaults to CWD)
sprocket format workflow.wdl                      # format in-place
sprocket format --check workflow.wdl              # format dry-run (exit 1 if changes needed)
sprocket validate --entrypoint my_workflow workflow.wdl inputs.json
sprocket analyzer                                 # start LSP server for IDE integration
```

---

## Interpreting Output

### womtool output

```
# Success
Success!

# Syntax error
ERROR: Unexpected symbol (line 4, col 5) when parsing '_gen4'.
  Expected rbrace, got input.
    input {
    ^

# Semantic error
ERROR: Task and namespace have the same name: Task defined here (line 3, col 6):
  task ps {
       ^
  Import statement defined here (line 1, col 20):
  import "ps.wdl" as ps
```

**womtool limitations to know:**
- Validates "can this be run on Cromwell" not just "is this spec-valid" — task-only files (no workflow) will fail older versions
- Does not enforce style or best-practice lint beyond type correctness
- WDL 1.2 support is partial; use sprocket for 1.2/1.3

---

### miniwdl check output

```
workflow.wdl (Ln 1, Col 1) MissingVersion
  document should declare WDL version; draft-2 assumed
task align (Ln 5, Col 17) CommandShellCheck, SC2086
  Double quote to prevent globbing and word splitting.
task align (Ln 12, Col 10) StringCoercion
  String s = :Int?:
task align (Ln 18, Col 14) NameCollision
  declaration of 'align' collides with a task name
task align (Ln 22, Col 10) UnusedDeclaration
  nothing references String scratch
```

Format: `file (Ln N, Col N) WarningName[, detail]`

**All miniwdl warning names:**

| Warning | Meaning |
|---------|---------|
| `MissingVersion` | No `version` statement → draft-2 assumed |
| `StringCoercion` | Implicit coercion to/from String (risky) |
| `UnusedDeclaration` | Variable declared but never referenced |
| `ForwardReference` | Variable referenced before its declaration |
| `NameCollision` | Declaration name shadows task/workflow name |
| `CommandShellCheck` | ShellCheck finding in command block (SC code appended) |
| `FileCoercion` | Implicit String→File coercion |
| `OptionalCoercion` | Non-optional used where optional expected |
| `NonemptyCoercion` | Array[T]+ used where Array[T] provided |
| `IncompleteCall` | Required inputs not supplied in call |
| `SelectArray` | select_first/select_all on non-optional array |
| `UnnecessaryQuantifier` | Quantifier that has no effect |
| `MixedIndentation` | Mixed tabs/spaces in command |

**Suppressing miniwdl warnings:**

```wdl
# Inline suppression — same line or the line after
task t {
    String s = i   # !ForwardReference !StringCoercion
    Int? i

    command <<<
        if [ ! -n ~{s} ]; then   # shellcheck: disable=SC2236
            echo Empty
        fi
    >>>

    output {
        String t = read_string(stdout())  # !NameCollision — meant to do that
    }
}
```

```bash
# Global suppression via CLI (not recommended — use inline instead)
miniwdl check --suppress ForwardReference,StringCoercion workflow.wdl
```

---

### sprocket output

```
error[v1::E001]: unknown type `Fiel`
  --> workflow.wdl:13:5
   |
13 |     Fiel outfile = "wc.txt"
   |     ^^^^ unknown type

warning[v1::W001]: missing `meta` section
  --> workflow.wdl:1:1
   |
 1 | version 1.2
   | ^^^^^^^^^^^ missing `meta` section in task `my_task`

note[v1::N001]: `input:` keyword is deprecated in WDL 1.2+
  --> workflow.wdl:20:10
```

Format: `level[code]: message` + source location span

**Sprocket exit codes:**
- `0` — no errors (warnings may still be present)
- `1` — one or more errors found
- `2` — internal tool error

---

## Sprocket Lint Rules

All rules are enabled with `sprocket lint` (or `sprocket check --lint`).

**Suppress a rule for the whole file (preamble directive):**
```wdl
#@ except: SnakeCase, DoubleQuotes
version 1.2
```

**Suppress a rule for a section:**
```wdl
#@ except: MetaSections
task legacy_task {
    command <<< echo hello >>>
}
```

### Full rule table

| Rule | Tags | What it checks |
|------|------|---------------|
| `CallInputKeyword` | Deprecated, Style | `input:` keyword in call bodies (deprecated in 1.2+) |
| `CommandSectionIndentation` | Spacing, Clarity | Mixed spaces/tabs in command section |
| `ConciseInput` | Style | Verbose input assignments where implicit binding is possible |
| `ConsistentNewlines` | Spacing, Portability | Mixed `\r\n` / `\n` newlines |
| `ContainerUri` | Clarity, Portability | Malformed or untagged container URIs in `runtime`/`requirements` |
| `DeclarationName` | Naming, Style | Names that redundantly include their type (e.g. `String string_val`) |
| `DeprecatedObject` | Deprecated | Use of deprecated `Object` type |
| `DeprecatedPlaceholder` | Deprecated | Deprecated `${...}` placeholder in heredoc commands |
| `DescriptionLength` | SprocketCompatibility | `meta.description` too long for Sprocket docs display |
| `DocCommentTabs` | Style, Clarity | Tab characters in doc comments (`##`) |
| `DocMetaStrings` | SprocketCompatibility | Reserved meta keys must have string values |
| `DoubleQuotes` | Style, Clarity | String literals using single quotes instead of double |
| `ExceptDirectiveValid` | Clarity, Correctness | Misplaced or misspelled `#@ except:` directives |
| `ExpectedRuntimeKeys` | Completeness, Deprecated | Missing expected keys in `runtime` section |
| `HereDocCommands` | Clarity, Correctness | Tasks using `command { }` instead of `command <<< >>>` |
| `ImportPlacement` | Clarity | Imports not placed after version, before other items |
| `InputName` | Naming, Style | Generic/too-short input names (`input`, `in`, `i`, `x`) |
| `InputSorted` | Sorting | Input declarations not sorted alphabetically |
| `KnownRules` | Correctness | Unknown rule IDs in `#@ except:` directives |
| `MatchingOutputMeta` | Completeness, Documentation | Output not documented in `meta.outputs` |
| `MetaDescription` | Completeness, Documentation | Missing `description` key in `meta` section |
| `MetaSections` | Completeness, Documentation | Missing `meta` or `parameter_meta` sections |
| `OutputName` | Naming, Style | Generic/too-short output names |
| `ParameterMetaMatched` | Completeness, Documentation | Input without matching `parameter_meta` entry |
| `PascalCase` | Naming, Style | Struct names not in PascalCase |
| `RedundantNone` | Style | `Optional? x = None` when `None` is already the default |
| `RequirementsSection` | Completeness, Portability | Missing `requirements` section in task (WDL 1.2+) |
| `RuntimeSection` | Completeness, Portability | Missing `runtime` section in task (WDL ≤1.1) |
| `SectionOrdering` | Style, Sorting | Sections not in spec-defined order |
| `ShellCheck` | Correctness | ShellCheck violations in command blocks |
| `SnakeCase` | Naming, Style | Tasks, workflows, variables not in `snake_case` |
| `TodoComment` | Style | TODO comments left in source |
| `UnusedDocComments` | Documentation | Doc comments (`##`) not attached to a supported element |

---

## Fix vs. Review Decision Guide

When Claude encounters WDL errors or warnings, use this classification:

### ✅ Auto-fix safe (deterministic, no semantic risk)

These can be applied without review:

| Issue | Tool | Fix |
|-------|------|-----|
| `MissingVersion` / no version statement | miniwdl, sprocket | Add `version 1.2` as first line |
| `DeprecatedPlaceholder` (`${var}` in heredoc) | sprocket | Replace `${var}` → `~{var}` throughout command |
| `DoubleQuotes` | sprocket | Replace `'string'` → `"string"` |
| `HereDocCommands` | sprocket | Convert `command { }` → `command <<< >>>` |
| `CallInputKeyword` (WDL 1.2+) | sprocket | Remove `input:` from call bodies |
| `DeprecatedObject` | sprocket | Replace `Object` type with a named `struct` |
| `SectionOrdering` | sprocket | Reorder sections to spec order: `input → private decls → command → output → requirements/runtime → meta → parameter_meta` |
| `ConsistentNewlines` | sprocket | Normalize to `\n` |
| `CommandSectionIndentation` | sprocket | Normalize indentation (tabs or spaces, not both) |
| `PascalCase` | sprocket | Rename struct to PascalCase |
| `SnakeCase` | sprocket | Rename task/workflow/variable to `snake_case` (confirm no callers first) |
| `RedundantNone` | sprocket | Remove `= None` from optional input declarations |
| Formatting (whitespace, alignment) | sprocket | Run `sprocket format workflow.wdl` |

---

### ⚠️ Review before fixing (may change behaviour or require decisions)

These need the user's input before applying:

| Issue | Tool | Why review needed |
|-------|------|------------------|
| `StringCoercion` | miniwdl | Implicit `Int?` → `String` coercion may be intentional; explicit cast or type change needed |
| `UnusedDeclaration` | miniwdl | Variable may be a placeholder or scaffold; check intent before deleting |
| `IncompleteCall` | miniwdl | Missing required inputs — must supply value or make input optional in task |
| `ForwardReference` | miniwdl | Re-ordering declarations may change semantics in some engines |
| `InputSorted` | sprocket | Alphabetical ordering is stylistic; confirm no tooling depends on declaration order |
| `MetaSections` | sprocket | Adding `meta`/`parameter_meta` requires writing descriptions — not auto-fillable |
| `ParameterMetaMatched` | sprocket | Requires writing meaningful documentation per input |
| `ContainerUri` | sprocket | Untagged images (`tool:latest`) need a specific digest or version tag |
| `ExpectedRuntimeKeys` | sprocket | Missing keys (`docker`, `memory`, `cpu`) need values appropriate to the workload |
| `RequirementsSection` | sprocket | Adding the section requires knowing resource requirements |
| `ShellCheck` / `CommandShellCheck` | both | Shell fix may change command behaviour — review each SC code |
| `DeclarationName` | sprocket | Renaming requires updating all references |
| `NameCollision` | miniwdl | Rename task or variable — scope implications need review |
| `ConciseInput` | sprocket | Implicit binding changes syntax style — confirm team prefers it |

---

## CI Integration Patterns

### Minimal CI (syntax errors only)
```bash
# Fast gate — catches parse/type errors, not style
miniwdl check --strict *.wdl
```

### Cromwell-targeted CI
```bash
# womtool for Cromwell semantic validation
for f in workflows/*.wdl; do
    java -jar womtool.jar validate "$f" || exit 1
done
```

### Full lint CI (recommended for new projects)
```bash
# sprocket covers 1.2/1.3 and has the strictest lint rules
sprocket lint .
sprocket format --check .    # fail if any file needs formatting
```

### Combined multi-tool CI
```bash
#!/usr/bin/env bash
set -euo pipefail

echo "=== miniwdl check ==="
miniwdl check --strict workflows/*.wdl tasks/*.wdl

echo "=== sprocket lint ==="
sprocket lint .

echo "=== sprocket format check ==="
sprocket format --check .

echo "All checks passed."
```

### GitHub Actions example
```yaml
name: WDL Lint
on: [push, pull_request]
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install miniwdl
        run: pip install miniwdl

      - name: Install shellcheck
        run: sudo apt-get install -y shellcheck

      - name: Install sprocket
        run: |
          curl -LO https://github.com/stjude-rust-labs/sprocket/releases/latest/download/sprocket-x86_64-unknown-linux-gnu.tar.gz
          tar -xzf sprocket-*.tar.gz
          sudo mv sprocket /usr/local/bin/

      - name: miniwdl check
        run: miniwdl check --strict **/*.wdl

      - name: sprocket lint
        run: sprocket lint .

      - name: sprocket format check
        run: sprocket format --check .
```

---

## Tool Selection by Situation

| Situation | Recommended tool(s) |
|-----------|-------------------|
| Targeting Cromwell/Terra | womtool validate **+** sprocket check |
| WDL 1.0 or 1.1 | womtool + miniwdl check (best 1.0/1.1 coverage) |
| WDL 1.2 or 1.3 | sprocket lint (womtool has partial 1.2 support only) |
| Command bash quality | miniwdl check (ShellCheck integration is most mature) |
| Style enforcement / formatting | sprocket lint + sprocket format |
| IDE integration | sprocket analyzer (LSP for VSCode, Neovim, etc.) |
| Quick syntax sanity check | miniwdl check (fastest, zero config) |
| Generate inputs JSON | womtool inputs |
| Task-only file (no workflow) | miniwdl check or sprocket (womtool may reject) |
| CI pipeline | sprocket lint + sprocket format --check |