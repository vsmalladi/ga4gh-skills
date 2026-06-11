---
name: ga4gh-reference-tools
description: Coordinate use of GA4GH reference genome and sequence tools (Refgenie, Refget, and their MCP servers). Trigger this skill whenever a user is working with reference data management, genome provisioning, sequence access, or needs guidance on which tool to use. Use this to select the right tool combination for bioinformatics workflows.
---

# GA4GH Reference Tools Coordination Skill

This skill helps you navigate and coordinate the GA4GH reference data ecosystem, including Refgenie (genome indexes/annotations) and Refget (sequence hashes/checksums).

## Tool Selection Matrix

Use this to pick the right tool for your task:

### I want to... → Use this tool

| Goal | Primary Tool | Secondary Tool | Notes | Skill |
|------|-------------|-----------------|-------|-------|
| **Search for genomes** | Refgenie MCP | - | No local install needed | [refgenie-mcp-skill](refgenie/mcp/SKILL.md) |
| **Download genome indexes** | Refgenie CLI | - | Full genome provisioning | [refgenie-cli-skill](refgenie/cli/SKILL.md) |
| **Find reference sequences** | Refget MCP | - | Discovery without local DB | [refget-mcp-skill](refget/mcp/SKILL.md) |
| **Download specific sequences** | Refget CLI | - | Individual sequence access | [refget-cli-skill](refget/cli/SKILL.md) |
| **Validate checksums** | Refget CLI | - | Verify downloaded data | [refget-cli-skill](refget/cli/SKILL.md) |
| **Setup alignment pipeline** | Refgenie CLI | Refget CLI | Indexes + sequences | [refgenie-cli-skill](refgenie/cli/SKILL.md) + [refget-cli-skill](refget/cli/SKILL.md) |
| **Browse collections** | Refgenie MCP + Refget MCP | - | Explore what's available | [refgenie-mcp-skill](refgenie/mcp/SKILL.md) + [refget-mcp-skill](refget/mcp/SKILL.md) |
| **Build custom DB** | Refget CLI | - | Local sequence database | [refget-cli-skill](refget/cli/SKILL.md) |

## Common Workflows

### Workflow 1: Setup Genome for Alignment

```
Step 1: Discover genome
  → Use Refgenie MCP: "What builds of human genome exist?"
  → See refgenie/mcp/SKILL.md
  
Step 2: Download indexes
  → Use Refgenie CLI: "Pull hg38 with BWA and bowtie2 indexes"
  → See refgenie/cli/SKILL.md
  
Step 3: Get sequences if needed
  → Use Refget MCP: "Find GRCh38 sequences"
  → See refget/mcp/SKILL.md
  → Use Refget CLI: "Download specific chromosome"
  → See refget/cli/SKILL.md
  
Result: Ready for alignment workflows
```

### Workflow 2: Validate Downloaded Data

```
Step 1: Have downloaded file
  → Use Refget CLI: "Get checksum"
  → See refget/cli/SKILL.md
  
Step 2: Compute file hash
  → Local: sha512sum file.fa
  
Step 3: Compare
  → Use Refget CLI: refget info for verification
  → See refget/cli/SKILL.md
  
Result: Confirm data integrity
```

### Workflow 3: Find & Download Genome Region

```
Step 1: Find genome
  → Use Refgenie MCP: List available genomes
  → See refgenie/mcp/SKILL.md
  
Step 2: Get sequences
  → Use Refget MCP: Find sequences in genome
  → See refget/mcp/SKILL.md
  
Step 3: Download region
  → Use Refget CLI: fetch with --start/--end
  → See refget/cli/SKILL.md
  
Result: Specific genomic region ready for analysis
```

### Workflow 4: Setup Local Reference Server

```
Step 1: Create database
  → Use Refget CLI: "Build local DB from FASTAs"
  → See refget/cli/SKILL.md
  
Step 2: Start server
  → Use Refget CLI: "refget serve --port 8000"
  → See refget/cli/SKILL.md
  
Step 3: Query from tools
  → Point your tools to http://localhost:8000
  
Result: Local reference service for your lab
```

## Decision Tree

```
START: "I need reference data"
│
├─ "I'm searching for what's available"
│  └─ Use MCP tools (no installation)
│     ├─ Refgenie MCP → Genome discovery
│     │  → See refgenie/mcp/SKILL.md
│     └─ Refget MCP → Sequence discovery
│        → See refget/mcp/SKILL.md
│
├─ "I'm downloading/installing locally"
│  └─ Use CLI tools (requires installation)
│     ├─ Refgenie CLI → Complete genome builds
│     │  → See refgenie/cli/SKILL.md
│     └─ Refget CLI → Individual sequences
│        → See refget/cli/SKILL.md
│
├─ "I need to validate data"
│  └─ Refget CLI → Checksum verification
│     → See refget/cli/SKILL.md
│
└─ "I need to integrate into pipeline"
   └─ Consider both:
      ├─ Refgenie (for indexes) → See refgenie/cli/SKILL.md
      └─ Refget (for sequences) → See refget/cli/SKILL.md
```

## Tool Comparison Quick Reference

### Refgenie
**Purpose**: Manage complete genome builds with pre-computed assets

**Provides**:
- FASTA sequences
- Alignment indexes (BWA, Bowtie2, etc.)
- Annotations
- Sizes/blacklists

**Best for**: Alignment workflows, complete genome provisioning

**Typical use**: 
```bash
refgenie pull -g hg38 -a fasta bwa_index
```

**Detailed guide**: See [refgenie/cli/SKILL.md](refgenie/cli/SKILL.md) for installation and usage

### Refget
**Purpose**: Access reference sequences via content-addressable hashes

**Provides**:
- Individual sequences
- Checksums (MD5, SHA512)
- Range queries
- Remote or local access

**Best for**: Sequence validation, targeted downloads, data integrity

**Typical use**:
```bash
refget fetch --hash sha512:... --start 1000 --end 2000
```

**Detailed guide**: See [refget/cli/SKILL.md](refget/cli/SKILL.md) for installation and usage

## Integration Examples

### Example 1: Download + Validate

```bash
# Step 1: Download via Refgenie
refgenie pull -g hg38 -a fasta

# Step 2: Get location
FASTA=$(refgenie seek -g hg38 -a fasta)

# Step 3: Get expected checksum from Refget
HASH=$(refget info --name "hg38" --json | jq .checksums.sha512)

# Step 4: Verify
ACTUAL=$(sha512sum $FASTA | cut -d' ' -f1)
[ "$ACTUAL" == "$HASH" ] && echo "✓ Valid" || echo "✗ Invalid"
```

### Example 2: Pipeline Setup

```bash
#!/bin/bash

# Get reference for alignment pipeline
GENOME="hg38"

# 1. Provision indexes
refgenie pull -g $GENOME -a fasta bwa_index bowtie2_index

# 2. Get paths
FASTA=$(refgenie seek -g $GENOME -a fasta)
BWA_INDEX=$(refgenie seek -g $GENOME -a bwa_index)

# 3. Use in pipeline
bwa mem $BWA_INDEX reads.fastq > aligned.sam
samtools view -T $FASTA aligned.sam > aligned.bam

# 4. Optionally validate specific regions with Refget
REGION_HASH="..."  # From Refget MCP query
refget fetch --hash $REGION_HASH > region.fa
```

### Example 3: Build Validation Network

```bash
# For multi-site validation

# Site 1: Host central Refget server
cd /shared/refget_db
refget serve --port 8000

# Site 2: Download and verify
refget fetch --hash sha512:... --server http://site1:8000

# Validate
sha512sum downloaded.fa
# Compare with known hash from Refget MCP
```

## Troubleshooting Guide

### "I don't know if a genome exists"
```
→ Use Refgenie MCP: refgenie_list_genomes()
→ Check spelling against returned list
```

### "I downloaded data, is it valid?"
```
→ Use Refget CLI: refget info --hash <HASH>
→ Compare: sha512sum file.fa vs. expected
```

### "I need indexes but only found sequences"
```
→ Use Refgenie CLI for complete builds
→ Or build indexes from sequences: bwa index, bowtie2-build
```

### "My tools can't find the reference"
```
→ Use Refgenie CLI: refgenie seek -g <GENOME> -a <ASSET>
→ Returns exact file path for your tools
```

### "I need to share references with team"
```
Option 1 (Shared Refgenie):
  → Setup shared Refgenie database
  → Point tools to refgenie.conf on shared drive

Option 2 (Refget Server):
  → Build local Refget DB
  → Start: refget serve --port 8000
  → Other machines connect to http://your-server:8000
```

## Performance Considerations

### Local vs. Remote

| Scenario | Recommendation |
|----------|-----------------|
| Single machine | Local Refgenie CLI |
| Small team | Shared Refgenie folder + Refget server |
| Large org | Dedicated reference server + MCP access |
| Cloud pipelines | Refget server + Refget CLI in containers |

### Caching Strategies

```bash
# Refgenie: Uses symlinks, very efficient
refgenie pull ...

# Refget: Use cache for repeated queries
export REFGET_CACHE_DIR=/tmp/refget_cache
refget fetch --hash ... --cache-dir $REFGET_CACHE_DIR
```

## MCP Server Configurations

The MCP servers referenced in this skill are implemented in the companion [ga4gh-mcp](https://github.com/ga4gh/ga4gh-mcp) repository:

### Reference Genome Servers
- **Refgenie MCP Server**: [servers/refgenie.py](https://github.com/ga4gh/ga4gh-mcp/blob/main/servers/refgenie.py)
  - Tools: `refgenie_*` functions for genome discovery and asset access
  - Default server: `http://refgenomes.databio.org`
  - Configuration: Environment variable `REFGENIE_URL`

- **Refget MCP Server**: [servers/ga4gh_refget.py](https://github.com/ga4gh/ga4gh-mcp/blob/main/servers/ga4gh_refget.py)
  - Tools: `refget_*` functions for sequence and collection access
  - Compatible with: seqcolapi.databio.org and other refget implementations

### Setup Instructions
- **Installation**: See [ga4gh-mcp README](https://github.com/ga4gh/ga4gh-mcp/blob/main/README.md#setup)
- **Claude Desktop Config**: Example configurations in [README](https://github.com/ga4gh/ga4gh-mcp/blob/main/README.md#claude-desktop--mcp-integration)
- **Testing**: Use [MCP Inspector](https://github.com/ga4gh/ga4gh-mcp/blob/main/README.md#testing-the-server) for development
- **Server Registry**: See [ga4gh-mcp-servers.json](ga4gh-mcp-servers.json) for complete server configuration details


## See Also

- **Individual Skills**:
  - [Refgenie MCP Skill](refgenie/mcp/SKILL.md) - Genome discovery
  - [Refgenie CLI Skill](refgenie/cli/SKILL.md) - Genome provisioning
  - [Refget MCP Skill](refget/mcp/SKILL.md) - Sequence discovery
  - [Refget CLI Skill](refget/cli/SKILL.md) - Sequence access
  
- **External Resources**:
  - GA4GH Standards: https://www.ga4gh.org/
  - Refgenie Docs: https://refgenie.org/
  - Refget Spec: https://github.com/ga4gh/refget