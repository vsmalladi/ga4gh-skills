# ga4gh-skills

## Project Goal

This repository provides AI agents with expert knowledge for GA4GH standards. Each skill contains code patterns, best practices, and examples that help agents generate correct, idiomatic code for common GA4GH tasks.

Target users range from developers integrating GA4GH services to researchers executing genomic analysis. 

## Requirements

### Python

* Python 3.9+

### CLI Tools


## Installation


## Skill Categories

| Category | Skills | Primary Tools | Description |
|----------|--------|---------------|-------------|
| workflow-management | x | WDL | Mangement of scalable analysis |
| reference-management | x | refget, refgenie | Mangement of genomic references and resources |


## Example Usage

Once skills are deployed, ask your agent naturally. Here are examples across common GA4GH workflows:

```
# Workflow Management

"Validate my CWL workflow before submission"
```

The agent will select appropriate tools based on context. See the Skill Categories table above for the complete list of available skills.

## Contributing

Key requirements:

* SKILL.md must include "Use when..." in description
* `primary_tool` must be a single value (not comma-separated)
* Quick Start uses bullets; Example Prompts use blockquotes
* Examples must document magic numbers with rationale
* Major multi-step code sections use Goal/Approach structure (intent survives version changes)
* Every Skill must include and `evals.json` to test wheathe the skill produces good outputs. 

## License / Links

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## AI Disclosure

Artificial intelligence tools, including large language models (LLMs), were used during the development of this project to support writing, clarify technical concepts, and assist in generating code snippets. These tools served as an aid for idea refinement, debugging, and improving the readability of explanations and documentation. All AI-generated text and code were thoroughly reviewed, verified for correctness, and understood in full before being incorporated into this work. The responsibility for all final decisions, interpretations, and implementations remains solely with the contributors.
