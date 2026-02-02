# Shadow-Clone

> Turn your code into a portfolio. Fast.

![Status: WIP](https://img.shields.io/badge/status-WIP-yellow)
![License: MIT](https://img.shields.io/badge/license-MIT-blue)

## What is this?

A Claude Code skill that generates portfolio-ready artifacts from your projects.
Built for vibe coders who ship fast and want to showcase their work without
becoming git experts.

## What it does

- **README Generation** - AI writes your project story with tech stack, trade-offs, and features
- **Architecture Diagrams** - Auto-generated C4 diagrams (Context, Container, Component)
- **Safe Code Pack** - Filtered, sanitized code extraction with secret redaction
- **Decision Logs** - Architecture Decision Records (ADRs) extracted from commits
- **Secret Scanning** - Detects API keys, credentials, tokens before publishing

## Who it's for

- **Vibe coders** - You ship with AI assistance and want to show recruiters what you built
- **Job hunters** - Quickly build a portfolio from your existing projects
- **Anyone** - Who wants professional project documentation without the hassle

## Installation

### Option 1: Copy to Claude Code Skills Directory

```bash
# Create skills directory if it doesn't exist
mkdir -p ~/.claude/skills

# Copy the skill file
cp skill/SKILL.md ~/.claude/skills/shadow-clone.md
```

### Option 2: Symlink (keeps skill updated with repo)

```bash
# Symlink to your clone of this repo
ln -s /path/to/Shadow-Clone/skill/SKILL.md ~/.claude/skills/shadow-clone.md
```

### Option 3: Manual Invocation

If you don't want to install, you can ask Claude Code to read and follow the skill:

```
Read skill/SKILL.md and follow its instructions for /path/to/my-project --readme-only
```

## Quick Start

After installation, use in any project:

```bash
# Generate portfolio README
/shadow-clone /path/to/your-project --readme-only

# Generate architecture diagrams
/shadow-clone /path/to/your-project --diagrams-only

# Extract safe code samples
/shadow-clone /path/to/your-project --safe-code-only

# Full pipeline (README + diagrams + ADRs + safe code)
/shadow-clone /path/to/your-project
```

### Output Structure

```
shadow-clone-output/
├── README.md                    # Portfolio-ready project narrative
├── .shadow-clone-meta.json      # Provenance metadata
├── diagrams/
│   ├── context.mermaid          # C4 Level 1: System Context
│   ├── containers.mermaid       # C4 Level 2: Containers
│   └── components.mermaid       # C4 Level 3: Components
├── docs/
│   ├── architecture.md          # Architecture overview with diagrams
│   └── decisions/               # Architecture Decision Records
│       ├── INDEX.md
│       └── 001-*.md
└── code/                        # Safe code pack (sanitized)
    └── ...
```

## Flags

| Flag | Purpose |
|------|---------|
| `--readme-only` | Generate README only |
| `--diagrams-only` | Generate architecture diagrams only |
| `--safe-code-only` | Extract safe code pack only |
| `--output DIR` | Output directory (default: `./shadow-clone-output/`) |
| `--mode conservative` | Maximum safety (default) |
| `--mode moderate` | Redact instead of exclude |
| `--rules "..."` | Natural language filtering rules |

## Safety Features

1. **Secret scanning** - Detects API keys, credentials, tokens, private keys
2. **Deny by default** - Nothing published without explicit allowlist
3. **Human review gate** - Always prompts before writing output
4. **Provenance tracking** - Records what was sanitized and why

## Design Document

See [docs/DESIGN.md](docs/DESIGN.md) for the full technical specification.

## Author

**Kyle Atekwana**

## License

MIT - see [LICENSE](LICENSE)
