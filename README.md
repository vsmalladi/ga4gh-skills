# ga4gh-skills

## Project Goal

This repository provides AI agents with expert knowledge for GA4GH standards. Each skill contains code patterns, best practices, and examples that help agents generate correct, idiomatic code for common GA4GH tasks,.

Target users range from developers integrating GA4GH services to researchers executing genomic analysis. 

## Requirements

### Python

• Python 3.9+

### CLI Tools


## Installation

### Claude Code

  git clone git@github.com:GA4GH/ga4gh-skills.git
  cd ga4gh-skills
  ./install-claude.sh                              # Install globally
  ./install-claude.sh --project /path/to/project   # Or install to specific project
  ./install-claude.sh --categories "workflow-mangement"  # Install specific categories
  ./install-claude.sh --list                       # List available skills
  ./install-claude.sh --validate                   # Validate all skills
  ./install-claude.sh --update                     # Only update changed skills
  ./install-claude.sh --uninstall                  # Remove all ga4gh-* skills

### Codex CLI

  ./install-codex.sh                               # Install globally
  ./install-codex.sh --project /path/to/project    # Or install to specific project
  ./install-codex.sh --categories "workflow-mangement"  # Install specific categories
  ./install-codex.sh --list                        # List available skills
  ./install-codex.sh --validate                    # Validate all skills
  ./install-codex.sh --update                      # Only update changed skills
  ./install-codex.sh --uninstall                   # Remove all ga4gh-* skills

### Gemini CLI

  ./install-gemini.sh                              # Install globally
  ./install-gemini.sh --project /path/to/project   # Or install to specific project
  ./install-gemini.sh --categories "workflow-mangement"  # Install specific categories
  ./install-gemini.sh --list                       # List available skills
  ./install-gemini.sh --validate                   # Validate all skills
  ./install-gemini.sh --update                     # Only update changed skills
  ./install-gemini.sh --uninstall                  # Remove all ga4gh-* skills

### OpenClaw

Install directly from [ClawHub](https://clawhub.ai/ga4gh/ga4gh-skills), or use the install script:

  ./install-openclaw.sh                            # Install all skills globally
  ./install-openclaw.sh --categories "workflow-mangement"  # Install specific categories
  ./install-openclaw.sh --project /path/to/workspace  # Install to workspace
  ./install-openclaw.sh --tool-type-metadata       # Add OpenClaw dependency metadata
  ./install-openclaw.sh --dry-run                  # Preview install + token estimate
  ./install-openclaw.sh --list                     # List available skills
  ./install-openclaw.sh --validate                 # Validate all skills
  ./install-openclaw.sh --update                   # Only update changed skills
  ./install-openclaw.sh --uninstall                # Remove all ga4gh-* skills

All installers support `--categories` for selective installation and `--dry-run` for previewing. Codex and Gemini convert to the Agent Skills standard (`examples/` -> `scripts/`, `usage-guide.md` -> `references/`). OpenClaw keeps the original directory structure and optionally adds dependency metadata with `--tool-type-metadata`.

## Skill Categories

| Category | Skills | Primary Tools | Description |
|----------|--------|---------------|-------------|
| workflow-management | x | Snakemake, Nextflow, CWL, WDL, Galaxy | Mangement of scalable analysis |

Total: 28 skills across 5 categories

## Example Usage

Once skills are deployed, ask your agent naturally. Here are examples across common GA4GH workflows:

```
# Workflow Management

"Validate my CWL workflow before submission"
```

The agent will select appropriate tools based on context. See the Skill Categories table above for the complete list of available skills.

## Contributing

Key requirements:

• SKILL.md must include "Use when..." in description
• `primary_tool` must be a single value (not comma-separated)
• Quick Start uses bullets; Example Prompts use blockquotes
• Examples must document magic numbers with rationale
• Major multi-step code sections use Goal/Approach structure (intent survives version changes)
• Every Skill must include and `evals.json` to test wheathe the skill produces good outputs. 

## License

MIT License - see LICENSE file for details.