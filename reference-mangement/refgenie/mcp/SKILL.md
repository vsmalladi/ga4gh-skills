---
name: refgenie-mcp
description: Use the Refgenie MCP (Model Context Protocol) server to search for and discover reference genome names and assets. Trigger this skill whenever a user wants to search for available reference genomes, browse genome collections, look up specific genome builds, or discover what reference assets are available without installing Refgenie locally. This is useful for exploratory queries, checking what references exist, or integrating genome discovery into workflows.
---

# Refgenie MCP Skill

This skill enables you to interact with the Refgenie MCP server to discover and search reference genome information.

## What This Skill Does

The Refgenie MCP server provides access to a searchable database of reference genomes and their associated assets (like indexes, sequences, and annotations). Use this when you need to:

- Search for available reference genome names
- Look up specific genome builds and versions
- Discover what asset types are available for a genome
- Browse the Refgenie collection without local installation

## When to Use This Skill vs. Refgenie CLI

- **Use Refgenie MCP** when: Searching for genomes, discovering what's available, integrating genome lookup into a workflow
- **Use Refgenie CLI** when: You need to actually download and manage reference genomes on your system

## Key Functions Available

### refgenie_list_genomes
Lists all available genomes in the Refgenie database.

```
Query the MCP with no parameters to get a comprehensive list of all reference genomes.
```

### refgenie_search_assets
Search for specific assets across genomes.

```
Provide:
- keyword: Search term (e.g., "fasta", "bwa", "bowtie2")
- genome_name (optional): Limit search to a specific genome
```

### refgenie_get_genome
Get detailed information about a specific genome.

```
Provide:
- genome_name: The genome identifier (e.g., "hg38", "mm10", "dm6")
Returns: Available assets and their details
```

### refgenie_list_assets
List all asset types available for a specific genome.

```
Provide:
- genome_name: The genome identifier
```

## Usage Patterns

### Discovery Pattern
```
User: "What reference genomes are available?"
→ Use refgenie_list_genomes to get the full catalog
→ Present the list organized by organism/build
```

### Search Pattern
```
User: "Do we have a BWA index for hg38?"
→ Use refgenie_get_genome("hg38")
→ or refgenie_search_assets("bwa", "hg38")
→ Report what's available and metadata (location, version)
```

### Asset Discovery Pattern
```
User: "What can I use with the mm10 genome?"
→ Use refgenie_list_assets("mm10")
→ Present organized by asset type (fasta, indexes, annotations)
```

## Integration Tips

- MCP calls are best for exploratory/discovery workflows
- Combine with Refgenie CLI skill when user wants to actually download genomes
- Use results to inform Refget queries (if needed to fetch specific sequences)
- Cache genome listings if the same genomes are queried multiple times in a conversation

## Common Reference Genome Names
- Humans: `hg38`, `hg19`, `GRCh38`, `GRCh37`
- Mice: `mm10`, `mm9`, `GRCm38`
- Drosophila: `dm6`, `dm3`
- Zebrafish: `danRer11`, `danRer10`
- Rat: `rn6`, `rn5`
- Arabidopsis: `TAIR10`

## Troubleshooting

**No results from search**: Try broader search terms or use `refgenie_list_genomes` to see available options

**Genome name not recognized**: Check spelling and case sensitivity; use list_genomes to verify the exact name

**Need to download**: Switch to the Refgenie CLI skill for actual download/installation tasks
