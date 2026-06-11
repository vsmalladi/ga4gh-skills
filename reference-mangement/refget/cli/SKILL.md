---
name: refget-cli
description: Install and use Refget CLI tools to download, query, and manage reference sequences locally. Trigger this skill whenever a user wants to install Refget (Python or Rust versions), download sequences by hash, query local sequence databases, validate sequence checksums, set up local Refget servers, or perform sequence retrieval workflows. Use for hands-on sequence access and validation.
---

# Refget CLI Skill

This skill covers installing and using Refget CLI tools to access reference sequences locally, including the official Python implementation and the Rust alternative (refget-rs).

## What is Refget?

Refget is a GA4GH standard for accessing reference sequences by cryptographic hash. It provides:
- Content-addressable sequence access (same sequence = same hash)
- Checksum verification (MD5, SHA512)
- Ranges queries (get specific regions)
- Standard HTTP API

## Installation Options

### Option 1: Python Version (Official)

#### Prerequisites
- Python 3.6+
- pip

#### Install
```bash
pip install refget
```

#### Verify
```bash
refget --version
refget --help
```

### Option 2: Rust Version (refget-rs)

#### Prerequisites
- Rust 1.70+ (install from https://rustup.rs/)

#### Install
```bash
cargo install refget
```

Or from GitHub:
```bash
git clone https://github.com/fg-labs/refget-rs.git
cd refget-rs
cargo install --path .
```

#### Verify
```bash
refget --version
refget --help
```

**Advantages of Rust version**: Faster performance, single binary, no dependency hell

## Core Commands

### Download Sequence by Hash

```bash
# Python version
refget fetch --hash sha512 <HASH>

# Rust version  
refget fetch <HASH>
```

**Example**:
```bash
refget fetch \
  sha512:6aeb0195f14c0d4a0b9a8d1e9f8d7c5b4a3b2c1d0e9f8d7c6b5a4f3e2d1c0b9a8d7e6f5g4h3i2j1
```

**Output**: Prints sequence to stdout or saves to file

### Query Sequence Range

```bash
refget fetch --hash sha512 <HASH> --start 1000 --end 2000
```

Returns only bases 1000-2000 (0-indexed, half-open)

### Get Sequence Metadata

```bash
refget info --hash sha512 <HASH>
```

Returns:
- Length
- MD5 checksum
- SHA512 checksum
- Aliases (if available)
- Supported ranges

### Verify Sequence Checksum

```bash
# Check if a sequence is in the database
refget info --hash sha512 <HASH> --quiet
echo $?  # 0 if found, 1 if not
```

Or with a file:
```bash
# Compute hash of local file
sha512sum file.fa

# Verify against remote
refget info --hash sha512 <COMPUTED_HASH>
```

## Local Database Setup

### Create Local Refget Database

```bash
# Initialize with a FASTA file
refget add --fasta genome.fa --name "My Genome"
```

### Query Local Database

```bash
# List sequences in your local database
refget list

# Get info on local sequence
refget info --hash md5 <YOUR_MD5>
```

### Serve Local Database

#### Python Version
```bash
refget server --port 8000
```

Server runs at `http://localhost:8000/sequence/{hash}`

#### Rust Version
```bash
refget serve --port 8000
```

## Practical Workflows

### Download Specific Genome Region

```bash
# Get hg38 chromosome 22
HASH="sha512:..."  # Get from Refget MCP query

refget fetch --hash $HASH --start 0 --end 50000000 > chr22.fa

# Verify
refget info --hash $HASH
```

### Validate Downloaded Sequence

```bash
# After downloading via other means
ACTUAL=$(sha512sum myfile.fa | cut -d' ' -f1)
EXPECTED="..."  # From Refget

if [ "$ACTUAL" == "$EXPECTED" ]; then
  echo "✓ Sequence validated"
else
  echo "✗ Checksum mismatch!"
fi
```

### Build Local Reference from Multiple Sequences

```bash
# Create directory for your reference
mkdir my_refget_db
cd my_refget_db

# Add sequences
refget add --fasta chr1.fa --name "Chr1"
refget add --fasta chr2.fa --name "Chr2"

# Start server
refget serve --port 8000 &

# Test
curl http://localhost:8000/sequence/{hash}?start=0&end=100
```

### Batch Download Sequences

```bash
#!/bin/bash

# List of hashes to download
HASHES=(
  "sha512:hash1"
  "sha512:hash2"
  "sha512:hash3"
)

for hash in "${HASHES[@]}"; do
  echo "Downloading $hash..."
  refget fetch --hash $hash > sequences/${hash}.fa
done
```

## Advanced Usage

### Set Custom Server

```bash
# Download from specific Refget server
refget fetch --hash $HASH --server https://refget.example.com
```

### Use with Alignment Tools

```bash
# Stream sequence to BWA
HASH="sha512:..."
refget fetch --hash $HASH | bwa mem -p - reads.fastq > aligned.sam
```

### Format Conversion

```bash
# Convert FASTA to 2bit format (if using local DB)
faToTwoBit genome.fa genome.2bit

# For Refget queries, use standard FASTA output
refget fetch --hash $HASH --output fasta
```

## Configuration

### Config File (~/.refgetrc or .refget.conf)

```yaml
# Default server
server: https://refget.example.com

# API token (if needed)
api_token: your_token_here

# Local database path
local_db: /path/to/local/refget/db

# Cache directory
cache_dir: /tmp/refget_cache
```

### Environment Variables

```bash
export REFGET_SERVER=https://refget.example.com
export REFGET_TOKEN=your_token
export REFGET_CACHE_DIR=/tmp/refget_cache
```

## Troubleshooting

### "Sequence not found"
```bash
# Verify hash format
# Should be: sha512:HEXSTRING or md5:HEXSTRING

# Try with different hash algorithm
refget info --hash md5 <HASH>
```

### "Connection refused"
- Check server is running: `refget serve --port 8000`
- Verify server URL is correct
- Check firewall/network

### Performance Issues
- Use Rust version (refget-rs) for better performance
- Add caching: `--cache-dir /tmp/refget_cache`
- Use `--start`/`--end` to fetch only needed regions

### Checksum Mismatch

```bash
# Verify download
sha512sum downloaded.fa

# Get expected hash from Refget
refget info --hash sha512 <HASH>

# If mismatch, re-download or check network
```

## Python vs Rust Implementation

| Feature | Python | Rust |
|---------|--------|------|
| Installation | `pip install refget` | `cargo install refget` |
| Performance | Good | Excellent |
| Memory | Higher | Lower |
| Dependencies | Python + packages | None (standalone) |
| Server | ✓ | ✓ |
| Scripting | Easy | Medium |

## Integration Tips

### With Refgenie
```bash
# Use Refget for specific sequences
# Use Refgenie for pre-built indexes

# Example: Get sequence + indexes for hg38
refgenie pull -g hg38 -a bwa_index
HASH=$(refget info --name "hg38")  # Get hash
refget fetch --hash $HASH --start 0 --end 1000000 > hg38_region.fa
```

### CI/CD Pipelines
```bash
# Download reference in container
docker run --rm -v /data:/data \
  refget fetch --hash sha512:... > /data/ref.fa
```

## See Also

- Refget Spec: https://github.com/ga4gh/refget
- refget-rs GitHub: https://github.com/fg-labs/refget-rs
- Refget MCP Skill: For discovering sequences
- Refgenie CLI Skill: For genome indexes
