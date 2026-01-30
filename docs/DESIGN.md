# Shadow-Clone - Claude Skill Design

**Date:** 2026-01-29
**Status:** Draft - Pending Implementation
**Purpose:** Transform private repositories into recruiter-ready portfolios while protecting IP

---

## Executive Summary

A Claude skill that generates sanitized, public-facing portfolio artifacts from private codebases. Helps developers showcase technical competence without exposing proprietary business logic, secrets, or IP.

### Key Design Decisions

- **Single skill with mode flags** (vs. skill family) - simpler mental model
- **Conservative deny-by-default** - only publish explicitly allowlisted content
- **Config file OR natural language rules** - flexibility for power users and quick runs
- **Local-first with review gate** - human approval before any publish
- **Granular targeting** - whole repo, specific folders, or individual files

---

## Skill Identity & Invocation

**Skill name:** `/shadow-clone`

### Basic Patterns

```bash
# Full pipeline (default)
/shadow-clone /path/to/repo

# With GitHub URL
/shadow-clone https://github.com/user/private-repo

# Specific modes
/shadow-clone /path/to/repo --readme-only
/shadow-clone /path/to/repo --diagrams-only
/shadow-clone /path/to/repo --safe-code-only

# Combined modes
/shadow-clone /path/to/repo --readme --diagrams

# With config file
/shadow-clone /path/to/repo --config .shadow-clone.yaml

# With natural language rules
/shadow-clone /path/to/repo --rules "exclude billing logic, redact company names, keep infrastructure code"

# Output location
/shadow-clone /path/to/repo --output ./my-portfolio

# Sensitivity override
/shadow-clone /path/to/repo --mode moderate
```

### Default Behavior

No flags = full pipeline with conservative mode, outputs to `./shadow-clone-output/`, prompts for review before publish.

---

## Granular Targeting (Files & Folders)

### Invocation Patterns

```bash
# Single file analysis
/shadow-clone /path/to/repo --file src/auth/oauth-handler.ts

# Single folder analysis
/shadow-clone /path/to/repo --folder src/infrastructure/

# Multiple targets
/shadow-clone /path/to/repo --file src/api/router.ts --folder src/utils/

# Glob patterns
/shadow-clone /path/to/repo --include "src/**/*.controller.ts"
```

### Output Per Target

| Target | Outputs |
|--------|---------|
| **Single file** | Summary (what it does, why it matters), sanitized snippet, dependency diagram (what it imports/exports), complexity notes |
| **Folder** | Folder-level README, internal architecture diagram, file manifest with role descriptions, representative snippets from key files |
| **Multiple targets** | All of the above, plus a relationship diagram showing how targets connect |

### Use Cases

- "I want to showcase just my auth system" → `--folder src/auth/`
- "This one file demonstrates my algorithm design skills" → `--file src/scoring/ranking-engine.ts`
- "Show my API design patterns" → `--include "**/*.controller.ts"`

---

## Configuration

### Option A: Config File (`.shadow-clone.yaml`)

```yaml
# .shadow-clone.yaml
project:
  name: "My Auth Service"
  role: "Lead Engineer"

safety:
  mode: conservative  # conservative | moderate

allow:
  paths:
    - src/infrastructure/**
    - src/utils/**
    - tests/**
    - .github/workflows/**
  patterns:
    - "*.dto.ts"
    - "*.interface.ts"

deny:
  paths:
    - src/billing/**
    - src/partners/**
  patterns:
    - "*secret*"
    - "*pricing*"

redact:
  company_names: true
  domain_names: true
  custom:
    - "AcmeCorp"
    - "internal.acme.io"

stubs:
  # Replace implementation with description
  - path: src/scoring/ranker.ts
    stub: "Proprietary ranking algorithm using [REDACTED] signals"

outputs:
  readme: true
  diagrams: true
  safe_code: true
```

### Option B: Natural Language Rules

```bash
/shadow-clone /path/to/repo --rules "
  keep all infrastructure and utility code,
  exclude anything in billing or partners folders,
  redact AcmeCorp and any internal domains,
  stub the ranking algorithm with a description
"
```

### How They Interact

- Config file = baseline rules (committed to repo, reusable)
- Natural language = overrides or quick one-off runs
- Can combine: `--config .shadow-clone.yaml --rules "also exclude the analytics folder"`

---

## Core Processing Pipeline

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   INGEST    │ ──▶ │   ANALYZE   │ ──▶ │  SANITIZE   │ ──▶ │  GENERATE   │ ──▶ │   REVIEW    │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
```

### Stage 1: Ingest

- Clone repo (if GitHub URL) or validate local path
- Load config file if present, parse natural language rules if provided
- Build file tree, detect languages, identify package structure

### Stage 2: Analyze

- Scan for sensitive patterns (secrets, credentials, PII)
- Classify files: infrastructure / utility / business logic / tests
- Map dependencies and imports between files
- Identify "showcase candidates" (well-structured, interesting code)
- Flag uncertainty: "This file might contain proprietary logic - needs review"

### Stage 3: Sanitize

- Apply allow/deny rules from config
- Redact company names, domains, identifiers
- Generate stubs for denied files that need representation
- Replace sensitive values with placeholders (`[REDACTED]`, `example.com`, `ACME_API_KEY`)

### Stage 4: Generate

- **README**: Project narrative, tech stack, your role, sanitized problem/solution
- **Diagrams**: Mermaid C4 diagrams from dependency analysis
- **Safe code pack**: Copy allowed files, insert stubs for denied ones
- **Per-file/folder artifacts**: Summaries, snippets, mini-diagrams (if targeted mode)

### Stage 5: Review Gate

- Output everything to local directory
- Generate `REVIEW.md` with: what's included, what's redacted, flagged uncertainties
- Prompt: "Review the output. Ready to publish to GitHub? (y/n)"

---

## Output Structure

```
shadow-clone-output/
├── README.md                    # Main project narrative
├── REVIEW.md                    # What's included/excluded, flags for human review
├── .shadow-clone-meta.json         # Provenance: source commit, config used, timestamp
│
├── docs/
│   ├── architecture.md          # C4 diagrams with explanations
│   ├── decisions/               # Generated ADRs (if enabled later)
│   │   └── 001-tech-stack.md
│   └── flows/                   # Sequence diagrams (if enabled later)
│       └── auth-flow.mermaid
│
├── diagrams/
│   ├── context.drawio.svg       # System context (what interacts with this)
│   ├── containers.drawio.svg    # Services, DBs, queues
│   └── components.drawio.svg    # Internal module structure
│
├── code/                        # Safe code pack
│   ├── infrastructure/
│   │   ├── terraform/
│   │   └── docker/
│   ├── utils/
│   │   └── validation.ts
│   ├── interfaces/
│   │   └── user.interface.ts
│   └── stubs/                   # Stubbed representations of excluded code
│       └── billing-service.stub.md
│
├── files/                       # Per-file artifacts (if --file used)
│   └── oauth-handler/
│       ├── summary.md
│       ├── snippet.ts
│       └── deps.mermaid
│
├── folders/                     # Per-folder artifacts (if --folder used)
│   └── auth/
│       ├── README.md
│       ├── architecture.mermaid
│       └── snippets/
│
└── index.md                     # Table of contents linking everything
```

### Provenance File (`.shadow-clone-meta.json`)

```json
{
  "source_repo": "github.com/user/private-repo",
  "source_commit": "a1b2c3d",
  "generated_at": "2026-01-29T10:30:00Z",
  "config_hash": "xyz789",
  "mode": "conservative",
  "tool_version": "1.0.0",
  "file_hashes": {
    "src/auth/oauth.ts": "abc123",
    "src/api/router.ts": "def456",
    "src/services/user.service.ts": "789ghi"
  },
  "diagram_sources": {
    "diagrams/context.drawio.svg": ["src/index.ts", "src/api/**"],
    "diagrams/containers.drawio.svg": ["src/services/**", "src/infrastructure/**"],
    "diagrams/components.drawio.svg": ["src/services/**"]
  },
  "diagram_hashes": {
    "diagrams/context.drawio.svg": "hash_at_generation",
    "diagrams/containers.drawio.svg": "hash_at_generation"
  }
}
```

**Delta Tracking Fields:**

| Field | Purpose |
|-------|---------|
| `file_hashes` | SHA hashes of source files at last generation |
| `diagram_sources` | Maps each diagram to the source files/globs that contribute to it |
| `diagram_hashes` | Hash of each diagram at generation time (for detecting manual edits) |

---

## Review Gate & Publish Flow

### Review Output

After generation completes, the skill presents:

```
═══════════════════════════════════════════════════════════
PORTFOLIO GENERATION COMPLETE
═══════════════════════════════════════════════════════════

Output location: ./shadow-clone-output/

INCLUDED:
  ✓ 12 files in code/infrastructure/
  ✓ 8 files in code/utils/
  ✓ 15 interface definitions
  ✓ 3 architecture diagrams

EXCLUDED (by config):
  ✗ src/billing/ (47 files)
  ✗ src/partners/ (23 files)

REDACTED:
  • "AcmeCorp" → "[COMPANY]" (34 occurrences)
  • "internal.acme.io" → "internal.example.com" (12 occurrences)

⚠️  FLAGS FOR REVIEW:
  1. src/utils/pricing-helper.ts - filename contains "pricing", included anyway
  2. src/auth/sso-config.ts - contains hardcoded URLs, replaced with placeholders
  3. code/stubs/ranking-service.stub.md - auto-generated stub, verify accuracy

═══════════════════════════════════════════════════════════

Next steps:
  1. Review output at ./shadow-clone-output/
  2. Check flagged items above
  3. Run: /shadow-clone --publish (when ready)
```

### Publish Destinations

```bash
# ═══════════════════════════════════════════════════════════
# OPTION 1: New GitHub Repository
# ═══════════════════════════════════════════════════════════
/shadow-clone --publish --new-repo my-portfolio
/shadow-clone --publish --new-repo my-portfolio --visibility public
/shadow-clone --publish --new-repo my-portfolio --visibility private
/shadow-clone --publish --new-repo my-portfolio --org my-company

# ═══════════════════════════════════════════════════════════
# OPTION 2: Fork from Original (creates sanitized fork)
# ═══════════════════════════════════════════════════════════
/shadow-clone --publish --fork my-project-public

# ═══════════════════════════════════════════════════════════
# OPTION 3: Branch in Same Repo
# ═══════════════════════════════════════════════════════════
/shadow-clone --publish --branch portfolio-public

# ═══════════════════════════════════════════════════════════
# OPTION 4: Subdirectory in Same Repo
# ═══════════════════════════════════════════════════════════
/shadow-clone --publish --subdir ./public-portfolio/

# ═══════════════════════════════════════════════════════════
# OPTION 5: Push to Existing Repo
# ═══════════════════════════════════════════════════════════
/shadow-clone --publish --existing user/my-existing-portfolio
/shadow-clone --publish --existing user/my-existing-portfolio --branch main

# ═══════════════════════════════════════════════════════════
# OPTION 6: Local Only (no GitHub)
# ═══════════════════════════════════════════════════════════
/shadow-clone --output ./shadow-clone-output --no-publish

# ═══════════════════════════════════════════════════════════
# OPTION 7: Export Package (zip/tarball)
# ═══════════════════════════════════════════════════════════
/shadow-clone --export ./my-portfolio.zip
/shadow-clone --export ./my-portfolio.tar.gz
```

### Destination Summary

| Destination | Flag | Creates | Best for |
|-------------|------|---------|----------|
| New repo | `--new-repo name` | Fresh GitHub repo | Clean public presence |
| Fork | `--fork name` | Sanitized fork | Showing "real" repo lineage |
| Branch | `--branch name` | Branch in source repo | Single repo, access control |
| Subdirectory | `--subdir path` | Folder in source repo | Monorepos |
| Existing repo | `--existing user/repo` | Commits to existing | Updating portfolio |
| Local only | `--no-publish` | Local files only | Manual control |
| Export | `--export path` | Zip/tarball | Direct sharing |

---

## Artifact Details

### README Generation

**Structure:**

```markdown
# Project Name

> One-line description of what this does and why it matters

## Overview

[AI-generated narrative: problem statement, constraints, solution approach]
[Your role and specific contributions]
[Key outcomes/impact - with sensitive metrics abstracted]

## Tech Stack

| Layer | Technologies |
|-------|--------------|
| Frontend | React, TypeScript, TailwindCSS |
| Backend | Node.js, Express, PostgreSQL |
| Infrastructure | Terraform, AWS (ECS, RDS, S3) |
| CI/CD | GitHub Actions |

## Architecture

[Embedded Mermaid diagram or link to docs/architecture.md]

## Key Features

- Feature 1: Brief description
- Feature 2: Brief description

## My Contributions

- Designed and implemented [specific system/component]
- Led [initiative] resulting in [abstracted outcome]

## Code Highlights

See [/code](/code) for sanitized examples of:
- Infrastructure as Code patterns
- API design and interfaces
- Testing strategies

## Trade-offs & Decisions

- Chose X over Y because Z
- Optimized for A at the cost of B

---

*Generated with shadow-clone from commit `a1b2c3d` on 2026-01-29*
```

**Options:**

```bash
--readme-style narrative    # Story-driven (default)
--readme-style technical    # Spec-heavy, less prose
--readme-style recruiter    # Quick-scan optimized, badges, highlights

--readme-sections "overview,tech,architecture,contributions"
--readme-skip "trade-offs"
--readme-inject intro.md    # Prepend your own intro
```

### Diagram Generation

**Types Generated:**

1. **Context Diagram (C4 Level 1)** - Who/what interacts with the system
2. **Container Diagram (C4 Level 2)** - Services, databases, queues
3. **Component Diagram (C4 Level 3)** - Internal module structure
4. **File/Folder Diagrams** - For targeted mode, shows dependencies

**Options:**

```bash
--diagram-format mermaid    # Default, GitHub-native
--diagram-format plantuml   # Alternative
--diagram-format drawio     # .drawio.svg - renders on GitHub, editable in draw.io
--diagram-format png        # Rendered images
--diagram-format svg        # Vector images (not editable)

--diagram-levels context              # Just C4 Level 1
--diagram-levels context,containers   # Levels 1-2
--diagram-levels all                  # Full C4 stack (default)

--diagram-theme default
--diagram-theme dark
--diagram-theme minimal
```

**Auto-detection Logic:**

- Scans imports/exports to map dependencies
- Identifies service boundaries from folder structure
- Detects infrastructure from config files (docker-compose, k8s, terraform)
- Infers data stores from ORM models and connection configs

### Draw.io SVG Generation

When using `--diagram-format drawio`, the skill generates `.drawio.svg` files - valid SVG images that render natively on GitHub while remaining fully editable in draw.io.

**How `.drawio.svg` Works:**

- Valid SVG markup that renders in browsers and GitHub markdown
- Embedded `<mxfile>` XML metadata containing draw.io diagram data
- Can be opened and edited in:
  - draw.io desktop app
  - draw.io web (diagrams.net)
  - VS Code with Draw.io Integration extension

**Generation Approach:**

Claude generates clean SVG markup with embedded draw.io metadata:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!-- Simplified example -->
<svg xmlns="http://www.w3.org/2000/svg" ...>
  <defs>
    <!-- draw.io metadata embedded here -->
    <mxfile host="shadow-clone">
      <diagram id="..." name="Context">
        <mxGraphModel>
          <!-- draw.io native format -->
        </mxGraphModel>
      </diagram>
    </mxfile>
  </defs>
  <!-- SVG content for rendering -->
  <rect ... />
  <text ... />
</svg>
```

**C4 Diagram Styling:**

- Minimal, clean aesthetic - simple rectangles and arrows
- Monochrome by default (works in light/dark themes)
- Clear labels with consistent typography
- No gradients or shadows - professional appearance

**Compatibility:**

| Tool | Support |
|------|---------|
| GitHub rendering | ✓ Displays as image |
| draw.io desktop | ✓ Full editing |
| diagrams.net web | ✓ Full editing |
| VS Code extension | ✓ Full editing |
| Browser direct | ✓ Displays as image |

### Delta Updates & Incremental Generation

The skill supports incremental updates to avoid regenerating unchanged diagrams and to preserve manual edits.

**Source Change Detection:**

- Compares source file hashes against stored values in `.shadow-clone-meta.json`
- Only re-analyzes files that have changed since last generation
- Tracks which source files contribute to each diagram

**Manual Edit Preservation:**

By default, the skill preserves diagrams that have been manually edited:

1. Detects manual edits by comparing diagram hash with last generated version
2. Skips regeneration if manual edits are detected
3. Warns user: "Skipping diagrams/context.drawio.svg - manual edits detected"

**Override Options:**

```bash
--force-diagrams             # Regenerate all diagrams, overwrite manual edits
--incremental                # Only process changed files (default: true)
--no-incremental             # Force full regeneration of everything
```

**Incremental Workflow:**

```
First run:
  → Analyzes all files
  → Generates all diagrams
  → Stores hashes in .shadow-clone-meta.json

Subsequent runs:
  → Compares file hashes
  → Identifies changed files
  → Regenerates only affected diagrams
  → Preserves manually-edited diagrams (unless --force-diagrams)
```

### Safe Code Pack

**Included by Default (Conservative Mode):**

| Category | Examples | Why it's safe |
|----------|----------|---------------|
| Infrastructure | Terraform, CloudFormation, Pulumi | Shows ops skills, rarely business logic |
| CI/CD | GitHub Actions, GitLab CI, Jenkinsfiles | Demonstrates automation |
| Containers | Dockerfiles, docker-compose, Helm, k8s | Standard patterns |
| Interfaces | TypeScript/Go interfaces, API contracts | Structure without implementation |
| DTOs/Models | Data transfer objects, request/response | Schema design skills |
| Utils/Helpers | Validation, formatting, retry logic | Reusable, not proprietary |
| Tests | Unit tests, integration test structure | Testing philosophy |
| Config | ESLint, Prettier, tsconfig, package.json | Quality signals |

**Stub Example:**

```typescript
// code/stubs/ranking-service.stub.ts

/**
 * STUB: RankingService
 *
 * Original: src/services/ranking/ranking.service.ts (847 lines)
 *
 * Purpose:
 * Calculates dynamic content ranking based on user engagement
 * signals and content freshness metrics.
 *
 * Key patterns used:
 * - Strategy pattern for pluggable ranking algorithms
 * - Caching layer with TTL-based invalidation
 * - Batch processing for performance
 *
 * Interfaces:
 */

export interface RankingInput {
  contentId: string;
  userId: string;
  context: RankingContext;
}

export interface IRankingService {
  calculateScore(input: RankingInput): Promise<RankingOutput>;
  batchCalculate(inputs: RankingInput[]): Promise<RankingOutput[]>;
}

// Implementation: [REDACTED - Proprietary Algorithm]
```

**Options:**

```bash
--include-tests              # Include test files (default: true)
--include-config             # Include config files (default: true)
--include-interfaces-only    # Just interfaces, no implementations

--stub-style minimal         # Just filename + one-liner
--stub-style summary         # Purpose + patterns (default)
--stub-style detailed        # Full interface extraction

--redact-comments            # Strip comments that might leak info
--normalize-names            # Replace internal naming conventions
--strip-todos                # Remove TODO/FIXME comments
```

---

## Complete Command Reference

```bash
/shadow-clone <source> [options]

SOURCE:
  /path/to/repo                    Local repository path
  https://github.com/user/repo     GitHub URL (clones temporarily)

MODES (what to generate):
  --full                           Full pipeline (default)
  --readme-only                    Just README
  --diagrams-only                  Just architecture diagrams
  --safe-code-only                 Just safe code pack
  --readme --diagrams              Combine specific modes

TARGETING (granular):
  --file <path>                    Single file analysis
  --folder <path>                  Single folder analysis
  --include "<glob>"               Pattern matching

CONFIGURATION:
  --config <file>                  Use config file (.shadow-clone.yaml)
  --rules "<natural language>"     Natural language rules
  --mode conservative|moderate     Safety posture (default: conservative)

OUTPUT:
  --output <path>                  Output directory (default: ./shadow-clone-output)
  --export <file.zip>              Export as archive

PUBLISH:
  --new-repo <name>                Create new GitHub repo
  --fork <name>                    Create sanitized fork
  --branch <name>                  Create branch in source repo
  --subdir <path>                  Create subdirectory in source repo
  --existing <user/repo>           Push to existing repo
  --visibility public|private      Repo visibility (default: private)
  --no-publish                     Local only, no GitHub
  --dry-run                        Preview publish without executing

CUSTOMIZATION:
  --readme-style narrative|technical|recruiter
  --diagram-format mermaid|plantuml|drawio|png|svg
  --diagram-levels context|containers|all
  --stub-style minimal|summary|detailed

INCREMENTAL/DELTA:
  --incremental                  Only process changed files (default: true)
  --no-incremental               Force full regeneration of everything
  --force-diagrams               Regenerate all diagrams, overwrite manual edits
```

---

## Typical Workflows

```bash
# Job hunting: full portfolio from private work repo
/shadow-clone ~/code/work-project --new-repo my-portfolio --visibility public

# Quick showcase: just one impressive module
/shadow-clone ~/code/project --folder src/auth/ --readme --diagrams

# Iterate locally before publishing
/shadow-clone ~/code/project --output ./draft --no-publish
# ... review and tweak ...
/shadow-clone --publish --existing user/my-portfolio

# Business idea protection: docs only, no code
/shadow-clone ~/code/startup-idea --readme-only --diagrams-only --new-repo startup-public
```

---

## Next Steps

1. **Create skill file** using `superpowers:writing-skills`
2. **Implement core pipeline** (analyze → sanitize → generate)
3. **Test with real repos** - iterate on sanitization accuracy
4. **Add publish integrations** (GitHub API)
5. **Refine natural language rule parsing**

---

## Appendix: Research References

This design is based on market research documented in:
- `shadow-clone-research-claude.md` - Comprehensive market research, MVP prioritization
- `shadow-clone-research-chatgpt.md` - Updated MVP scope, safe code pack approach
- `shadow-clone-research-gemini.md` - Technical architecture, competitive analysis

Key insights incorporated:
- Recruiters spend 6-37 seconds on GitHub profiles
- Conservative deny-by-default is critical for trust
- Human-in-the-loop review is non-negotiable
- Infrastructure/IaC code is high-signal, low-risk to share
