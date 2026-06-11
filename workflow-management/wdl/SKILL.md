---
name: wdl
description: >
  Write, understand, and structure Workflow Description Language (WDL) files.
  Use this skill whenever the user is writing or editing WDL tasks, workflows.
  Trigger on phrases like "cromwell", "miniwdl", "Terra workflow", "WDL runtime block",
  or any request to define bioinformatics or data-processing pipelines using
  WDL syntax. Also trigger when the user pastes WDL code and asks for help,
  review, or explanation.
---

# WDL — Workflow Description Language

**Spec reference:** WDL 1.3 (current) — [SPEC.md](https://github.com/openwdl/wdl/blob/wdl-1.3/SPEC.md) | [OpenWDL Docs](https://docs.openwdl.org)
All previous versions: [1.2](https://github.com/openwdl/wdl/blob/wdl-1.2/SPEC.md) · [1.1](https://github.com/openwdl/wdl/blob/wdl-1.1/SPEC.md) · [1.0](https://github.com/openwdl/wdl/blob/legacy/versions/1.0/SPEC.md) · [draft-2](https://github.com/openwdl/wdl/blob/legacy/versions/draft-2/SPEC.md)

All examples below target `version 1.2` or `version 1.3` unless otherwise noted.
Use `version 1.2` for broadest engine compatibility; use `version 1.3` for
`else`/`else if`, enums, and dynamic retry logic.

> For deeper detail on spec-defined behaviour (type coercion, scoping rules,
> standard library functions) see `references/wdl-spec-highlights.md`.
> For linting and fixing WDL, see `lint/SKILL.md`.

---

## Document Structure

Every WDL file follows this top-level order:

```
version 1.2          ← required, must be first non-comment line

import "..."         ← optional imports

struct Foo { ... }   ← optional struct definitions (global scope)

task my_task { ... } ← one or more tasks

workflow my_wf { ... } ← zero or one workflow per file
```

---

## Task Anatomy

A task is the atomic unit of computation — a Bash command plus metadata.

```wdl
version 1.2

task align_reads {
    # ── INPUTS ────────────────────────────────────────────────────────
    input {
        File   reads_1
        File   reads_2
        File   reference
        String sample_id
        Int    threads    = 8      # optional input with default
        Int    memory_gb  = 16
    }

    # ── PRIVATE DECLARATIONS (computed, not inputs) ────────────────────
    Int disk_gb = ceil(
        size(reads_1, "GB") + size(reads_2, "GB") +
        size(reference, "GB") * 2
    ) + 50

    # ── COMMAND ───────────────────────────────────────────────────────
    command <<<
        set -euo pipefail
        bwa mem \
            -t ~{threads} \
            -R "@RG\tID:~{sample_id}\tSM:~{sample_id}" \
            ~{reference} ~{reads_1} ~{reads_2} \
        | samtools sort -@ ~{threads} -o ~{sample_id}.sorted.bam
        samtools index ~{sample_id}.sorted.bam
    >>>

    # ── OUTPUTS ───────────────────────────────────────────────────────
    output {
        File bam  = "~{sample_id}.sorted.bam"
        File bai  = "~{sample_id}.sorted.bam.bai"
    }

    # ── REQUIREMENTS (preferred in 1.2+; replaces runtime in 1.3) ─────
    requirements {
        container:  "biocontainers/bwa-samtools:latest"
        cpu:        threads
        memory:     "~{memory_gb} GB"
        disks:      "local-disk ~{disk_gb} HDD"
        preemptible: 2
        maxRetries:  1
    }

    # ── METADATA ──────────────────────────────────────────────────────
    meta {
        description: "Align paired-end reads with BWA-MEM and sort with samtools"
        author:      "Your Name"
    }

    parameter_meta {
        reads_1:    { description: "R1 FASTQ (gzipped)" }
        reads_2:    { description: "R2 FASTQ (gzipped)" }
        reference:  { description: "BWA-indexed reference FASTA" }
        sample_id:  { description: "Used to name output files" }
        bam:        { description: "Coordinate-sorted BAM" }
        bai:        { description: "BAM index" }
    }
}
```

**Key rules:**
- `command <<<  >>>` is the heredoc form — prefer it over `command { }`. Interpolation uses `~{expr}`, not `${expr}`.
- Only `command` is required. All other sections are optional but recommended.
- `requirements` is the WDL 1.2+ preferred spelling of `runtime`. Use `runtime` only for 1.0 compatibility.
- Private declarations between `input` and `command` are computed, not user-supplied.

---

## Workflow Anatomy

```wdl
version 1.2

import "tasks/align.wdl"  as align
import "tasks/qc.wdl"     as qc

workflow dna_pipeline {
    input {
        Array[File] reads_1_files
        Array[File] reads_2_files
        Array[String] sample_ids
        File   reference
        Boolean run_qc = true
    }

    scatter (idx in range(length(sample_ids))) {
        call qc.fastqc {
            input:
                fastq   = reads_1_files[idx],
                threads = 4
        }

        call align.align_reads {
            input:
                reads_1   = reads_1_files[idx],
                reads_2   = reads_2_files[idx],
                reference = reference,
                sample_id = sample_ids[idx]
        }
    }

    output {
        Array[File] bams        = align_reads.bam
        Array[File] qc_reports  = fastqc.html
    }

    meta {
        description: "Scatter-gather DNA alignment pipeline"
    }
}
```

**Key rules:**
- One workflow per WDL file.
- `call` brings a task's outputs into scope as `task_name.output_name`.
- Inside a `scatter`, outputs become `Array[T]` automatically.
- Import aliases (`as align`) prevent namespace collisions.

---

## Types

| Category   | Types |
|------------|-------|
| Primitive  | `String`, `Int`, `Float`, `Boolean` |
| File types | `File`, `Directory` |
| Optional   | `String?`, `File?`, etc. — value may be `None` |
| Compound   | `Array[T]`, `Map[K,V]`, `Pair[L,R]`, `Object`, `enum` |
| Non-empty  | `Array[T]+` — must have ≥1 element |
| User-defined | `struct MyType { ... }` |

**Optional handling:**

```wdl
# Provide a fallback with select_first
String name = select_first([optional_name, "default"])

# Test if defined
if (defined(optional_file)) {
    call process { input: f = select_first([optional_file]) }
}

# Optional output from conditional block
File? result = if_block_task.output_file   # automatically File?
```

---

## Scatter and Gather

```wdl
# Scatter over an array
scatter (sample in samples) {
    call process { input: s = sample }
}
# process.result is now Array[File]

# Scatter with index (zip two arrays)
scatter (idx in range(length(ids))) {
    call run {
        input:
            id   = ids[idx],
            file = files[idx]
    }
}

# Nested scatter
scatter (batch in batches) {
    scatter (sample in batch) {
        call analyse { input: s = sample }
    }
    # analyse.result is Array[File] here
}
# Outside: Array[Array[File]]
```

---

## Conditionals

**WDL 1.0–1.2 (if only):**

```wdl
if (run_qc) {
    call fastqc { input: fastq = reads }
}
File? qc_html = fastqc.html   # optional because block may not run
```

**WDL 1.3 (if / else if / else):**

```wdl
if (mode == "fast") {
    call quick_align { input: reads = reads }
} else if (mode == "sensitive") {
    call sensitive_align { input: reads = reads }
} else {
    call default_align { input: reads = reads }
}

File bam = select_first([
    quick_align.bam,
    sensitive_align.bam,
    default_align.bam
])
```

---

## Structs

```wdl
version 1.2

struct ReferenceBundle {
    File   fasta
    File   fai
    File   dict
    File?  known_sites_vcf   # optional member
}

workflow variant_call {
    input {
        ReferenceBundle ref
        Array[File]     bam_files
    }

    scatter (bam in bam_files) {
        call haplotype_caller {
            input:
                bam      = bam,
                ref_fasta = ref.fasta,
                ref_dict  = ref.dict
        }
    }
}
```

Structs are declared at file scope (not inside tasks or workflows) and are globally accessible within the file.

---

## Enums (WDL 1.3)

```wdl
version 1.3

enum Aligner {
    BWA
    Bowtie2
    STAR
}

workflow align {
    input {
        Aligner aligner = BWA
    }

    if (aligner == BWA) {
        call bwa_mem { ... }
    } else if (aligner == Bowtie2) {
        call bowtie2 { ... }
    }
}
```

---

## Dynamic Retry / Resource Escalation (WDL 1.3)

```wdl
version 1.3

task memory_hungry {
    input { File big_file }

    command <<< run_tool ~{big_file} >>>

    requirements {
        memory:     "~{8 * task.attempt} GB"   # doubles each retry
        maxRetries: 3
    }
}
```

`task.attempt` starts at 1. `task.previous.memory` gives the prior attempt's value.

---

## String Interpolation and Expressions

```wdl
# Basic interpolation
String out = "~{sample_id}.bam"

# Conditional expression
String flag = if paired then "--paired" else ""

# Sep for arrays
command <<<
    tool --inputs ~{sep="," input_files}
>>>

# Arithmetic
Int disk = ceil(size(bam, "GB") * 3) + 20

# String functions
String base = basename(fastq, ".fastq.gz")
String dir  = dirname(bam)
```

---

## Imports and Subworkflows

```wdl
version 1.2

# Import with alias (recommended)
import "https://raw.githubusercontent.com/org/repo/main/tasks/qc.wdl" as qc
import "tasks/align.wdl"    as align
import "workflows/joint_genotyping.wdl" as jg

workflow main {
    call qc.run_fastqc { ... }
    call align.bwa_mem { ... }
    call jg.joint_genotype { ... }   # calling a subworkflow
}
```

---

## Input JSON

```json
{
    "my_workflow.sample_ids":    ["sample1", "sample2"],
    "my_workflow.reads_1_files": ["gs://bucket/s1_R1.fq.gz",
                                  "gs://bucket/s2_R1.fq.gz"],
    "my_workflow.reference":     "gs://bucket/ref/hg38.fa",
    "my_workflow.run_qc":        true
}
```

Key format: `workflow_name.input_name`. Generate a template with:

```bash
womtool inputs workflow.wdl > inputs.json
miniwdl run --empty-inputs workflow.wdl
```

---

## Run Commands

```bash
# Validate / check syntax
womtool validate workflow.wdl
miniwdl check workflow.wdl
sprocket check workflow.wdl

# Generate inputs template
womtool inputs workflow.wdl
sprocket inputs workflow.wdl

# Local execution
java -jar cromwell.jar run workflow.wdl -i inputs.json
miniwdl run workflow.wdl -i inputs.json
sprocket run workflow.wdl inputs.json
toil-wdl-runner workflow.wdl --input inputs.json

# Local execution with options file (Cromwell)
java -jar cromwell.jar run workflow.wdl \
    -i inputs.json \
    -o cromwell-options.json \
    -p workflow_root.zip
```

---

## Common Patterns

**Calculate disk from input file sizes:**
```wdl
Int disk_gb = ceil(size(bam, "GB") * 3 + size(reference, "GB")) + 20
```

**Read a file line-by-line in output:**
```wdl
output {
    Array[String] lines = read_lines("output.txt")
    Array[File]   bams  = glob("*.bam")
    Map[String,String] kv = read_map("metadata.tsv")
}
```

**Pass optional flags:**
```wdl
command <<<
    tool \
        ~{if defined(bed_file) then "--bed " + select_first([bed_file]) else ""} \
        --input ~{bam}
>>>
```

---

## Execution Engines Quick Reference

| Engine    | Version support | Best for |
|-----------|-----------------|----------|
| Cromwell  | 1.0 – 1.1       | Google Cloud, AWS Batch, HPC, Terra |
| miniwdl   | 1.0 – 1.1       | Local development, Docker/Singularity |
| toil      | 1.0 – 1.1       | CWL runner that can also run WDL |
| Sprocket  | 1.0 – 1.3       | First full 1.2/1.3 compliant engine |
| Terra/AnVIL | 1.0 – 1.1     | NIHBroad cloud platform (Cromwell backend) |


---

## Sub-Skills

- **`lint/SKILL.md`** — Validate, lint, and fix WDL files using `miniwdl check` and `womtool validate`. Covers auto-fixable issues vs. issues that need review.

## Spec Reference Files

- **`references/wdl-spec-highlights.md`** — Cross-version condensed spec: type system, stdlib by version, scoping rules, requirements/runtime/hints sections, and pitfall table covering draft-2 through 1.3. Read when answering precise language or compatibility questions.