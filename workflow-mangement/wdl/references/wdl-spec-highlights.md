# WDL Spec Highlights — All Versions

Condensed cross-version reference. For full authoritative detail:
- WDL 1.3 spec: https://github.com/openwdl/wdl/blob/wdl-1.3/SPEC.md
- WDL 1.2 spec: https://github.com/openwdl/wdl/blob/wdl-1.2/SPEC.md
- WDL 1.1 spec: https://github.com/openwdl/wdl/blob/wdl-1.1/SPEC.md
- WDL 1.0 spec: https://github.com/openwdl/wdl/blob/legacy/versions/1.0/SPEC.md
- WDL draft-2 spec: https://github.com/openwdl/wdl/blob/legacy/versions/draft-2/SPEC.md
- Upgrade guide: https://docs.openwdl.org/reference/upgrade-guide.html

---

## Table of Contents

1. [Version History & What Changed](#1-version-history--what-changed)
2. [Versioning Rules](#2-versioning-rules)
3. [Type System (all versions)](#3-type-system-all-versions)
4. [Type Coercion](#4-type-coercion)
5. [Scoping Rules](#5-scoping-rules)
6. [Command Section](#6-command-section)
7. [Standard Library — by version](#7-standard-library--by-version)
8. [Requirements / Runtime / Hints Sections](#8-requirements--runtime--hints-sections)
9. [Task Variable (1.2+)](#9-task-variable-12)
10. [File Path Semantics](#10-file-path-semantics)
11. [Imports](#11-imports)
12. [Common Pitfalls & Version Traps](#12-common-pitfalls--version-traps)

---

## 1. Version History & What Changed

### draft-2 (no `version` statement — legacy)
The original WDL. Files with **no `version` statement** are treated as draft-2 by all engines.

Key characteristics:
- Interpolation in both `command { }` and `command <<< >>>` uses `${...}` syntax
- No `input { }` section — all declarations at the top of a task are treated as inputs
- No `output { }` section type safety — wildcard glob (`*`) allowed directly
- `struct` does not exist — use `Object` for compound types
- No `scatter` nesting support in Cromwell
- `true`/`false` placeholder option: only one side required (other defaults to empty string)
- Optional unspecified variables evaluate to the empty string (not an error)
- `sep`, `true`/`false`, `default` placeholder options exist but differ from 1.0+

```wdl
# draft-2 style — no version statement, no input{} block
task align {
    File reads
    Int threads = 4

    command {
        bwa mem -t ${threads} ${reads} > out.bam
    }

    output {
        File bam = "out.bam"
    }

    runtime {
        docker: "biocontainers/bwa:latest"
        memory: "8G"
        cpu: threads
    }
}
```

---

### WDL 1.0 — first versioned release
Added `version 1.0` statement. Introduced structured `input { }` and `output { }` blocks.

**New in 1.0 vs draft-2:**
- `version 1.0` statement required (first non-comment line)
- Explicit `input { }` section in tasks and workflows
- `struct` type introduced — replaces `Object` for typed compound data
- Heredoc command `command <<< >>>` must use `~{...}` placeholders (not `${...}`)
- `true`/`false` placeholder: both sides now required
- `None` keyword for optional values
- `scatter` nesting supported
- Wildcard glob in output section dropped
- `meta` and `parameter_meta` sections can contain numbers, booleans, objects, and arrays (not just strings)
- Subworkflow output section required when called as a sub-workflow

```wdl
version 1.0

struct ReferenceBundle {
    File fasta
    File fai
    File dict
}

task align {
    input {
        File reads
        ReferenceBundle ref
        Int threads = 4
    }

    command <<<
        bwa mem -t ~{threads} ~{ref.fasta} ~{reads} > out.bam
    >>>

    output {
        File bam = "out.bam"
    }

    runtime {
        docker: "biocontainers/bwa:latest"
        memory: "8 GB"
        cpu: threads
    }
}
```

---

### WDL 1.1 — hidden types, None formalized
**New in 1.1 vs 1.0:**
- `None` is now formally typed as the hidden type `Union` (previously informal)
- `Object` type deprecated (will be removed in 2.0) — use `struct` instead
- `struct` aliasing in imports: `import "x.wdl" as x alias Foo as Bar`
- `file://` protocol for imports deprecated
- Standard library reorganized hierarchically
- `select_first` gains an optional `default` parameter
- `length()` generalized to also accept `Map`, `Object`, and `String`
- `keys(Struct|Object)` variant added
- `read_tsv` gains parameters to read header rows and return `Array[Object]`
- Clarifications: accessing a non-existent struct/object/call member is an error

```wdl
version 1.1

import "tasks/qc.wdl" as qc alias SampleInfo as QCSampleInfo

workflow example {
    input {
        File? optional_bed   # None if not provided
    }

    String bed_arg = if defined(optional_bed)
        then "--bed " + select_first([optional_bed])
        else ""
}
```

---

### WDL 1.2 — requirements/hints, Directory type, stdlib expansion
**New in 1.2 vs 1.1:**

**Language:**
- `Directory` type — semantically distinct from `File` for directory paths
- `requirements { }` section replaces `runtime { }` (runtime deprecated, removed in 2.0)
- `hints { }` section for non-binding engine suggestions (separate from hard requirements)
- Workflow `hints { }` section added
- `task` scoped variable in `requirements`/`hints` — `task.attempt`, `task.name`
- Multi-line strings via `<<<  >>>` syntax in declarations (not just command)
- `input:` keyword in `call` blocks is now optional (was required boilerplate)
- Exponentiation operator (`**`)
- Struct `meta` and `parameter_meta` sections
- Relaxed struct coercion — extra keys in Map→Struct coercion are allowed and ignored
- Struct-to-struct conversion allowed when types are compatible
- `Array[T]+` non-empty array enforced

**New standard library functions (1.2):**
- `contains_key(map_or_object, key)` → `Boolean`
- `values(map)` → `Array[V]`
- `find(string, pattern)` → `String?`
- `matches(string, pattern)` → `Boolean`
- `chunk(array, size)` → `Array[Array[T]]`
- `join_paths(path1, path2, ...)` → `String`
- `contains(array, value)` → `Boolean`
- `size()` generalized to accept any compound value

```wdl
version 1.2

task process {
    input {
        Directory ref_dir   # new Directory type
        File bam
        Int memory_gb = 8
    }

    String output_name = <<<
        ~{basename(bam, ".bam")}_processed
    >>>

    command <<<
        run_tool \
            --ref-dir ~{ref_dir} \
            --bam ~{bam} \
            --out ~{output_name}
    >>>

    requirements {                    # replaces runtime
        container: "tool:latest"
        cpu: 4
        memory: "~{memory_gb} GB"
        disks: "local-disk 100 HDD"
    }

    hints {                           # non-binding suggestions
        disks: "local-disk 200 SSD"
        gpu: false
    }
}

workflow my_wf {
    input { File bam }

    # input: keyword is now optional in call blocks
    call process {
        bam = bam
    }

    hints {
        allow_nested_inputs: true
    }
}
```

---

### WDL 1.3 — enums, else/else-if, split(), dynamic retry
**New in 1.3 vs 1.2:**

**Language:**
- `enum` type — closed set of named variants with associated values
- `else if` / `else` conditional clauses in workflows (previously required two negated `if`s)
- `task.previous` — access prior attempt's computed requirements
- `task.max_retries` — total max retries configured for the task
- Clarified path existence semantics (see §10)

**New standard library functions (1.3):**
- `split(string, delimiter)` → `Array[String]`
- `value(enum_variant)` → inner value of an enum

```wdl
version 1.3

enum Aligner {
    BWA   = "bwa"
    STAR  = "STAR"
    Bowtie2 = "bowtie2"
}

workflow align {
    input {
        Aligner aligner = Aligner.BWA
        String  mode    = "fast"
    }

    # else if / else — new in 1.3
    if (mode == "fast") {
        call quick_align { aligner_bin = value(aligner) }
    } else if (mode == "sensitive") {
        call deep_align  { aligner_bin = value(aligner) }
    } else {
        call default_align { aligner_bin = value(aligner) }
    }
}

task memory_hungry {
    input { File big_file }

    command <<< run_tool ~{big_file} >>>

    requirements {
        # Doubles memory on each retry; cap at 128 GB
        memory:      "~{min(task.previous.memory * 2, 128)} GB"
        max_retries: 4
    }
}
```

---

## 2. Versioning Rules

- Language version is two-number (`1.3`); spec version is three-number (`1.3.0`)
- No `version` statement → engine must treat file as **draft-2**
- Minor version bump = additive, non-breaking
- Major version bump = breaking changes (draft-2 → 1.0 was effectively a major break)
- All files in a workflow must use the same major version
- A higher minor version file **can** import lower minor version files: `1.3` can import `1.2`, `1.1`, `1.0`
- A lower minor version file **cannot** import higher: `1.1` cannot import `1.2`
- Current stable: **1.3**. In-development breaking version: **2.0**

---

## 3. Type System (all versions)

### Primitive types (all versions)
| Type      | Introduced | Notes |
|-----------|-----------|-------|
| `String`  | draft-2   | Unicode, single or double quotes |
| `Int`     | draft-2   | 64-bit signed integer |
| `Float`   | draft-2   | 64-bit IEEE 754 |
| `Boolean` | draft-2   | `true` / `false` literals |
| `File`    | draft-2   | Path; must exist at eval time (clarified in 1.3) |
| `Directory` | 1.2     | Directory path; must exist at eval time |

### Compound types
| Type           | Introduced | Notes |
|----------------|-----------|-------|
| `Array[T]`     | draft-2   | Ordered, may be empty |
| `Array[T]+`    | 1.2       | Non-empty enforcement |
| `Map[K, V]`    | draft-2   | Keys must be primitive |
| `Pair[L, R]`   | draft-2   | `.left` / `.right` accessors |
| `Object`       | draft-2   | **Deprecated 1.1**, hidden in 2.0 — use `struct` |
| `struct Name { ... }` | 1.0 | User-defined; global scope |
| `enum Name { ... }` | 1.3 | Closed set with associated values |

### Optional types
- `T?` — value may be `None`
- `None` — special untyped null; hidden type `Union`
- Multi-level optionals (`Int??`) are not allowed
- Nested optionals within compound types are allowed: `Array[String?]?`

---

## 4. Type Coercion

**Automatic (implicit):**
- `Int` → `Float`
- `String` → `File`
- `T` → `T?`
- `Array[T]` → `Array[T]?`
- `Map[String, X]` → `Struct` with matching member names (1.2: extra keys allowed and ignored)
- `Object` → `Struct` (1.0–1.1 only; use struct literals in 1.2+)

**No other implicit coercions.** Use stdlib functions for explicit conversion:
- `as_pairs(map)` → `Array[Pair[K,V]]`
- `as_map(pairs)` → `Map[K,V]`
- `keys(struct_or_object)` → `Array[String]` (1.1+)

---

## 5. Scoping Rules

- **Task scope**: `input` → private declarations → `command` → `output` → `requirements`/`runtime` → `meta` → `parameter_meta`
- **Workflow scope**: `input` → calls/scatter/conditionals/declarations → `output` → `meta` → `parameter_meta`
- **Scatter**: new inner scope; outer identifiers accessible; inner identifiers invisible outside; outputs automatically become `Array[T]`
- **Conditional `if`**: new inner scope; outputs become `T?` outside; `else`/`else if` outputs remain `T?` (1.3)
- **Structs**: file-level scope; globally accessible within the file
- **Imported names**: accessible via alias namespace only

**Name resolution order:**
1. Local scope (innermost block first)
2. Enclosing scopes outward
3. Call outputs (`call_name.output_name`)
4. Import namespaces (`alias.name`)

No forward references — declarations must precede use.

---

## 6. Command Section

Two syntaxes — **prefer heredoc:**

```wdl
# Heredoc (preferred, 1.0+) — ~{} interpolation
command <<<
    set -euo pipefail
    tool --input ~{input_file} --threads ~{threads}
>>>

# Brace form (draft-2 / legacy) — ${} interpolation
command {
    tool --input ${input_file}
}
```

**In `command <<< >>>` only `~{expr}` is valid.** The `${...}` syntax is reserved for the brace form.

### Placeholder options (inline modifiers)

| Option | Syntax | Available |
|--------|--------|-----------|
| Array join | `~{sep=", " array_var}` | draft-2+ |
| Boolean flag | `~{true="--flag" false="" bool_var}` | draft-2+ (both sides required in 1.0+) |
| Optional default | `~{default="none" opt_var}` | draft-2+ |

### Special outputs

```wdl
output {
    File out      = stdout()    # captures stdout
    File err_log  = stderr()    # captures stderr
    Array[File] results = glob("*.bam")   # wildcard match in exec dir
}
```

---

## 7. Standard Library — by version

### Available in all versions (draft-2+)

**File/String:**
| Function | Returns | Notes |
|----------|---------|-------|
| `basename(f)` | `String` | Filename only |
| `basename(f, suffix)` | `String` | Strip suffix too |
| `dirname(f)` | `String` | Directory portion |
| `size(f, unit)` | `Float` | "B","KB","MB","GB","TB" |
| `glob(pattern)` | `Array[File]` | Only valid in task output section |
| `read_string(f)` | `String` | Whole file, trimmed |
| `read_int(f)` | `Int` | First line |
| `read_float(f)` | `Float` | First line |
| `read_boolean(f)` | `Boolean` | First line |
| `read_lines(f)` | `Array[String]` | One per line |
| `read_tsv(f)` | `Array[Array[String]]` | Two-dimensional |
| `read_map(f)` | `Map[String,String]` | Two-column TSV |
| `read_json(f)` | `Union` | JSON → WDL |
| `write_lines(a)` | `File` | Array → temp file |
| `write_tsv(a)` | `File` | 2D array |
| `write_map(m)` | `File` | Map as TSV |
| `write_json(v)` | `File` | Any value as JSON |
| `stdout()` | `File` | Task stdout |
| `stderr()` | `File` | Task stderr |

**Math:**
| Function | Notes |
|----------|-------|
| `ceil(f)` | → `Int`, round up |
| `floor(f)` | → `Int`, round down |
| `round(f)` | → `Int`, nearest |
| `min(a, b)` | Same type |
| `max(a, b)` | Same type |

**Array/Map:**
| Function | Returns |
|----------|---------|
| `length(a)` | `Int` |
| `range(n)` | `Array[Int]` — `[0..n-1]` |
| `zip(a, b)` | `Array[Pair[A,B]]` |
| `unzip(pairs)` | `Pair[Array[L], Array[R]]` |
| `cross(a, b)` | `Array[Pair[A,B]]` (cartesian) |
| `flatten(aa)` | `Array[T]` (one level) |
| `transpose(aa)` | `Array[Array[T]]` |
| `keys(map)` | `Array[K]` |
| `as_pairs(m)` | `Array[Pair[K,V]]` |
| `as_map(pairs)` | `Map[K,V]` |
| `select_first([...])` | `T` — first non-None |
| `select_all([...])` | `Array[T]` — all non-None |
| `defined(x)` | `Boolean` |

**String:**
| Function | Notes |
|----------|-------|
| `sub(s, pattern, replace)` | Regex substitution |
| `sep(delim, array)` | Join string array |

---

### Added in WDL 1.1

- `select_first([..., default])` — optional `default` parameter added
- `length()` — now also accepts `Map`, `Object`, `String`
- `keys(Struct|Object)` → `Array[String]` variant (member names)
- `read_tsv(f, header, fields)` — gains optional header-row and field-name params, can return `Array[Object]`
- `abs(n)` — absolute value (some engines had this earlier; 1.1 formalizes it)

---

### Added in WDL 1.2

| Function | Returns | Notes |
|----------|---------|-------|
| `contains_key(map_or_obj, key)` | `Boolean` | Key existence check |
| `values(map)` | `Array[V]` | Map values |
| `find(string, pattern)` | `String?` | First regex match, or None |
| `matches(string, pattern)` | `Boolean` | Full-string regex match |
| `chunk(array, size)` | `Array[Array[T]]` | Split array into sub-arrays |
| `join_paths(p1, p2, ...)` | `String` | Path joining |
| `contains(array, value)` | `Boolean` | Array membership |
| `size()` | `Float` | Now accepts any compound value |

---

### Added in WDL 1.3

| Function | Returns | Notes |
|----------|---------|-------|
| `split(string, delimiter)` | `Array[String]` | String splitting |
| `value(enum_variant)` | inner type | Extract enum's associated value |

---

## 8. Requirements / Runtime / Hints Sections

### draft-2 / 1.0 / 1.1 — `runtime` (only section)

```wdl
runtime {
    docker:  "image:tag"
    memory:  "8 GB"        # or "8G" in draft-2
    cpu:     4
    disks:   "local-disk 100 HDD"
    preemptible: 2
    maxRetries:  1
}
```

Common Cromwell-specific keys (still valid in runtime for backward compatibility):

| Key | Notes |
|-----|-------|
| `docker` | Cromwell's image key (`container` in 1.2+) |
| `memory` | String: `"N GB"`, `"N MB"`, etc. |
| `cpu` | Int or Float |
| `disks` | `"local-disk <GB> HDD|SSD"` |
| `preemptible` | Int — GCP preemptible VM attempts |
| `maxRetries` | Int — Cromwell spelling |
| `bootDiskSizeGb` | Int — GCP boot disk |
| `zones` | String — GCP zone list |
| `noAddress` | Boolean — disable external IP |

---

### WDL 1.2+ — `requirements` + `hints`

`runtime` is **deprecated in 1.2** and will be **removed in 2.0**.

**`requirements` — binding constraints (engine must satisfy):**

| Key | Type | Notes |
|-----|------|-------|
| `container` | `String` or `Array[String]` | OCI image URI |
| `cpu` | `Int` or `Float` | vCPUs |
| `memory` | `String` | `"N GB"` |
| `disks` | `String` or `Array[String]` | `"local-disk N HDD"` or `"N GB"` |
| `gpu` | `Boolean` | GPU required |
| `fpga` | `Boolean` | FPGA required |
| `max_retries` | `Int` | (note underscore — spec spelling) |
| `return_codes` | `Int` or `Array[Int]` | Exit codes treated as success |

**`hints` — non-binding suggestions:**

| Key | Type | Notes |
|-----|------|-------|
| `disks` | `String` | Preferred disk type/size |
| `gpu` | `Boolean` or object | GPU preference |
| `fpga` | `Boolean` | FPGA preference |
| `max_cpu` | `Int` | Upper bound on CPU |
| `max_memory` | `String` | Upper bound on memory |

> **Cromwell compatibility note:** Cromwell (as of 87) still uses `runtime` keys. When targeting Cromwell/Terra, use `runtime` or check current Cromwell docs for `requirements` support status.

---

## 9. Task Variable (1.2+)

Available inside `requirements`, `hints`, and `command` sections only.

| Variable | Available | Notes |
|----------|-----------|-------|
| `task.attempt` | 1.2+ | Current attempt number (starts at 1) |
| `task.name` | 1.2+ | Task name as a string |
| `task.previous` | 1.3+ | Prior attempt's computed requirements object |
| `task.max_retries` | 1.3+ | Configured max retries |

```wdl
# 1.2: linear memory escalation
requirements {
    memory:      "~{8 * task.attempt} GB"
    max_retries: 3
}

# 1.3: exponential escalation with cap using task.previous
requirements {
    memory:      "~{min(task.previous.memory * 2, 128)} GB"
    max_retries: 4
}
```

---

## 10. File Path Semantics

Clarified in WDL 1.3 (applies to all versions conceptually):

- **Outside `output` section** (input, private declarations, command): relative paths resolve relative to the **WDL document's parent directory**
- **Inside `output` section**: relative paths resolve relative to the **task's execution directory**
- `File` and `Directory` values must exist at declaration evaluation time
- `File?` / `Directory?` evaluate to `None` if path does not exist (do not error)
- Input files/directories are **read-only** (clarified in 1.2)

---

## 11. Imports

```wdl
# Basic import (uses file name as namespace)
import "tasks/align.wdl"

# With alias (recommended to avoid collisions)
import "tasks/qc.wdl" as qc

# Struct alias (1.1+) — rename an imported struct
import "tasks/types.wdl" as t alias SampleInfo as TypesSampleInfo

# Remote import (engine-dependent)
import "https://raw.githubusercontent.com/org/repo/main/tasks/qc.wdl" as qc
```

Rules:
- All files in a workflow must share the same **major version**
- Higher minor version can import lower: `1.3` → `1.2` ✓; `1.1` → `1.2` ✗
- `file://` protocol deprecated in 1.1
- Structs from imports accessible as `alias.StructName`

---

## 12. Common Pitfalls & Version Traps

| Pitfall | Detail | Fix |
|---------|--------|-----|
| `${var}` in heredoc | Only valid in brace-form command | Use `~{var}` in `command <<< >>>` |
| Missing `version` | Engine treats file as draft-2 | Add `version 1.x` as first line |
| `runtime` in 1.2+ | Deprecated; may be removed by engines targeting 2.0 | Migrate to `requirements` + `hints` |
| `max_retries` vs `maxRetries` | Spec key is `max_retries` (underscore); Cromwell uses `maxRetries` (camelCase) | Match to your engine |
| `container` vs `docker` | Spec 1.2+ key is `container`; Cromwell runtime uses `docker` | Match to your engine |
| Optional output used directly | `T?` from conditional cannot be used where `T` required | Wrap in `select_first([...])` or guard with `defined()` |
| Scatter output type | Inside scatter: `T`. Outside: `Array[T]`. Inside conditional: `T?` | Be explicit about expected types |
| Disk undersized | Large inputs need multiplier + buffer | `ceil(size(f, "GB") * 3 + size(ref, "GB")) + 20` |
| `glob()` outside output | Only valid inside task `output { }` block | Move to output section |
| `Object` usage in 1.1+ | Deprecated — use `struct` | Define a named struct instead |
| `Array[T]+` not enforced in 1.0/1.1 | Only runtime-enforced in 1.2+ | Add a guard `if (length(arr) == 0)` for older versions |
| Same major version across imports | Cannot mix 1.x and 2.x | Keep all files on same major version |
| `input:` required in call blocks | Required in 1.0/1.1; optional in 1.2+ | Add `input:` for max compatibility |
| Struct assignment from non-literal | Deprecated in 1.1 (Map→Struct coercion was ambiguous) | Use struct literal syntax explicitly |
