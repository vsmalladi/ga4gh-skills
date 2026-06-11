---
name: refgenie-cli
description: Install and use Refgenie CLI to download and manage reference genomes locally. Trigger this skill whenever a user wants to install Refgenie, download reference genome assets, manage a local Refgenie database, configure Refgenie settings, or perform CLI-based genome management tasks. Use this for hands-on genome provisioning workflows.
---

# Refgenie CLI Skill

This skill covers installing Refgenie and using its command-line interface to download and manage reference genomes on your system.

## Installation

### Prerequisites
- Python 3.6+
- pip (Python package manager)
- ~50GB+ available disk space (depending on genomes you download)

### Install Refgenie

```bash
pip install refgenie
```

### Verify Installation
```bash
refgenie --version
refgenie --help
```

## Configuration

### Initialize Refgenie
Before using Refgenie, initialize a database directory:

```bash
refgenie init -c /path/to/refgenie.conf
```

This creates a config file and default database structure. 

**Tip**: Use a path on a drive with plenty of free space, as genomes can be large.

### Set Default Configuration
```bash
export REFGENIE=/path/to/refgenie.conf
```

Or add to `.bashrc`/`.zshrc` for persistence.

## Common Commands

### List Available Genomes
```bash
refgenie list -r
```
Shows all available genomes in the remote repository.

### List Downloaded Genomes
```bash
refgenie list
```
Shows genomes you've already downloaded locally.

### Download a Genome

```bash
refgenie pull -g hg38 -a fasta bwa_index
```

**Parameters**:
- `-g` / `--genome`: Genome name (e.g., `hg38`, `mm10`, `dm6`)
- `-a` / `--asset`: Specific assets to download (space-separated)

**Common assets**:
- `fasta` - Raw genome sequence
- `bwa_index` - BWA alignment index
- `bowtie2_index` - Bowtie2 alignment index
- `fai_index` - Samtools fasta index
- `chrom_sizes` - Chromosome sizes
- `blacklist` - ENCODE blacklist regions
- `anno_feature` - Annotations

### Pull All Assets for a Genome
```bash
refgenie pull -g hg38
```

### Get Asset Location
```bash
refgenie seek -g hg38 -a fasta
```
Returns the file path to the downloaded asset.

### View Asset Metadata
```bash
refgenie info -g hg38 -a bwa_index
```

## Practical Workflows

### Setup Complete Genome for Alignment
```bash
# Initialize
refgenie init -c ~/.refgenie/hg38.conf

# Download key assets
refgenie pull -g hg38 -a fasta bwa_index samtools_index chrom_sizes

# Verify
refgenie seek -g hg38 -a bwa_index
```

### Download Multiple Genomes
```bash
for genome in hg38 mm10 dm6; do
  refgenie pull -g $genome -a fasta bowtie2_index
done
```

### Check Disk Usage
```bash
# List downloaded genomes with sizes
du -sh /path/to/refgenie/*/

# Get specific genome size
du -sh /path/to/refgenie/hg38/
```

## Advanced Options

### Build Custom Assets
```bash
refgenie build -g hg38 -a custom_asset
```

### Update Configuration
```bash
refgenie config -c /path/to/refgenie.conf
refgenie config set genome_folder /new/path
```

### Remote Server (for shared installations)
```bash
refgenie pull -g hg38 -s http://refgenomes.databio.org
```

## Troubleshooting

### "Genome not found"
- Verify genome name: `refgenie list -r | grep hg38`
- Check spelling and case sensitivity

### "Asset not available"
- List available assets: `refgenie list -g hg38 -a`
- Asset might need to be built first

### Slow Download / Network Issues
- Check connection: `ping refgenomes.databio.org`
- Try partial download: `refgenie pull -g hg38 -a fasta` (instead of all assets)
- Retry: Downloads are resumable

### "No space left on device"
- Check disk space: `df -h`
- Clean unused genomes: `rm -rf /path/to/refgenie/old_genome/`

## Integration with Other Tools

### Using with BWA
```bash
# Get path to BWA index
BWA_INDEX=$(refgenie seek -g hg38 -a bwa_index)
bwa mem $BWA_INDEX reads.fastq > aligned.sam
```

### Using with Samtools
```bash
# Get FASTA and index
FASTA=$(refgenie seek -g hg38 -a fasta)
FAI=$(refgenie seek -g hg38 -a fai_index)
samtools view -h -T $FASTA aligned.bam | ...
```

## See Also

- Refgenie Official Docs: https://refgenie.org/
- Refgenie MCP Skill: For discovering genomes without local installation
- Refget Skills: For fetching specific sequences by hash
