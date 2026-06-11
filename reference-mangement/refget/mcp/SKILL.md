---
name: refget-mcp
description: Use the Refget MCP (Model Context Protocol) server to search and access reference sequences by hash. Trigger this skill whenever a user wants to search for sequence collections, look up sequences by identifier, verify sequence checksums, browse Refget collections, or integrate sequence access into workflows. Ideal for discovering what sequences are available without local installation.
---

# Refget MCP Skill

This skill enables you to interact with the Refget MCP server to discover and access reference sequences identified by cryptographic hashes.

## What This Skill Does

Refget is a standard GA4GH protocol for accessing reference sequences. The MCP server provides queryable access to sequence collections. Use this when you need to:

- Search available reference sequence collections
- Look up sequence metadata by accession or name
- Verify sequence checksums (MD5, SHA512)
- Discover sequence availability across collections
- Integrate sequence lookup into bioinformatics workflows

## When to Use This Skill vs. Refget CLI

- **Use Refget MCP** when: Searching collections, discovering sequences, checking what's available
- **Use Refget CLI** when: You need to actually download and query sequences locally

## Key Differences from Refgenie

| Aspect | Refgenie | Refget |
|--------|----------|--------|
| **Purpose** | Manages pre-built genome indexes & annotations | Provides access to reference sequences by hash |
| **Use Case** | Alignment, mapping workflows | Sequence retrieval, validation, distribution |
| **Scope** | Complete genome builds | Individual sequences across collections |
| **Assets** | Indexes, alignments, annotations | Sequences, metadata, checksums |

## Key Functions Available

### refget_collections
List all available reference sequence collections.

```
Query the MCP with no parameters to discover all sequence collections.
Returns: Collection identifiers, descriptions, and metadata
```

### refget_get_collection
Get detailed information about a specific collection.

```
Provide:
- collection_id: The collection identifier
Returns: Sequences in collection, metadata, URI endpoints
```

## Usage Patterns

### Discovery Pattern
```
User: "What reference sequences are available?"
→ Use refget_collections to list all collections
→ Present organized by organism/collection type
```

### Collection Search Pattern
```
User: "Show me sequences in the GRCh38 collection"
→ Use refget_get_collection("GRCh38")
→ List sequences, accessions, and checksums
```

### Sequence Lookup Pattern
```
User: "What's available for chromosome 22?"
→ Query collections for chr22 entries
→ Return sequence IDs and download URIs
```

## Common Reference Collections

### Human
- `GRCh38` / `hg38` - GRC Human Build 38
- `GRCh37` / `hg19` - GRC Human Build 37

### Mouse
- `GRCm38` / `mm10` - GRC Mouse Build 38
- `GRCm39` / `mm39` - GRC Mouse Build 39

### Other Organisms
- `Arabidopsis thaliana` - TAIR10
- `Drosophila melanogaster` - dm6
- `Danio rerio` - GRCz11
- `Saccharomyces cerevisiae` - R64

## Sequence Access

### Get Sequence by Hash
Once you have a sequence hash from a collection, you can retrieve the actual sequence using:

```
Refget URI format:
https://refget.server/sequence/{hash}?start={start}&end={end}
```

**Parameters**:
- `hash`: The sequence hash (MD5 or SHA512)
- `start` (optional): Start position
- `end` (optional): End position

### Checksum Verification
```
The MCP returns checksums (MD5, SHA512) for sequence validation
Use to verify downloaded sequences match expected values
```

## Integration Patterns

### With Refgenie
```
User wants full genome build:
→ Use Refgenie for indexes/annotations
→ Use Refget for specific sequences if needed
```

### Sequence Validation
```
Downloaded a FASTA file?
→ Use Refget MCP to get expected checksums
→ Verify: md5sum file.fa && compare with Refget
```

### Bioinformatics Pipelines
```
Pipeline needs reference sequences:
→ Query Refget to find collection
→ Get sequence hash/checksum
→ Download if needed via Refget CLI
→ Validate against Refget checksums
```

## Troubleshooting

**Collection not found**: Try variations of the name (e.g., "GRCh38" vs "hg38")

**No sequences in collection**: Verify collection exists and is not empty

**Need actual sequences**: Switch to Refget CLI for downloading

**Checksum mismatch**: Sequences may have been modified; verify collection version

## See Also

- Refget Official Spec: https://github.com/ga4gh/refget
- Refgenie Skills: For genome indexes and annotations
- Refget CLI Skill: For downloading and querying sequences locally
- GA4GH Standards: https://www.ga4gh.org/
