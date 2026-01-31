---
name: shadow-clone
description: Use when creating quick portfolios from codebases, summarizing projects for easier digest, or generating shareable project documentation
---

# Shadow-Clone

Create quick portfolios and code summaries for easier digest.

## Quick Reference

| Flag | Purpose |
|------|---------|
| `--readme-only` | Generate README only |
| `--diagrams-only` | Generate architecture diagrams only |
| `--safe-code-only` | Extract safe code pack only |
| `--config FILE` | Use config file |
| `--rules "..."` | Natural language rules |
| `--output DIR` | Output directory (default: `./shadow-clone-output/`) |
| `--mode MODE` | `conservative` (default) or `moderate` |
| `--file PATH` | Target specific file |
| `--folder PATH` | Target specific folder |
| `--include GLOB` | Include by glob pattern |

## Invocation

```bash
# Full pipeline (default)
/shadow-clone /path/to/repo

# Specific outputs
/shadow-clone /path/to/repo --readme-only
/shadow-clone /path/to/repo --diagrams-only
/shadow-clone /path/to/repo --safe-code-only

# With natural language rules
/shadow-clone /path/to/repo --rules "exclude billing logic, keep infrastructure code"

# Target specific code
/shadow-clone /path/to/repo --folder src/infrastructure/
```

## Workflow

```dot
digraph shadow_clone_workflow {
    rankdir=TB;
    node [shape=box];

    start [label="Parse arguments" shape=ellipse];
    validate [label="Validate target path"];
    ingest [label="Stage 1: Ingest\n(scan repo structure)"];
    analyze [label="Stage 2: Analyze\n(detect sensitive content)"];
    sanitize [label="Stage 3: Sanitize\n(apply rules)"];
    generate [label="Stage 4: Generate\n(create artifacts)"];
    review [label="Stage 5: Review Gate\n(human approval)"];
    output [label="Write to output directory" shape=ellipse];

    start -> validate;
    validate -> ingest;
    ingest -> analyze;
    analyze -> sanitize;
    sanitize -> generate;
    generate -> review;
    review -> output;
}
```

## Stage Details

### Stage 1: Ingest
- Scan repository structure
- Build file tree
- Detect project type (language, framework)
- Load config file if present

### Stage 2: Analyze
- **Secret scanning** - API keys, credentials, tokens
- **IP detection** - Proprietary algorithms, business logic
- **PII scanning** - Names, emails, internal URLs
- Build sensitivity map per file

### Stage 3: Sanitize
- Apply allow/deny rules
- Redact sensitive content
- Generate placeholder replacements

### Stage 4: Generate
Based on mode flags, generate:
- **README** - Project overview, tech stack, architecture summary
- **Diagrams** - Mermaid C4 diagrams (Context, Container, Component levels)
- **Safe Code Pack** - Allowlisted files with sanitization applied

### Stage 5: Review Gate
- Display summary of what will be published
- Show any warnings or flagged content
- **Require explicit user approval before output**

## Output Structure

```
shadow-clone-output/
├── README.md
├── docs/
│   ├── architecture.md
│   └── flows/
├── public/           # Safe code pack
│   ├── infrastructure/
│   └── utils/
└── .shadow-clone-meta.json  # Provenance
```

## Safety Principles

1. **Deny by default** - Nothing published without explicit allowlist
2. **Human in the loop** - Always prompt for review before output
3. **Provenance tracking** - Record what was sanitized and why
4. **Conservative mode** - Default to maximum safety

## README Generation (`--readme-only`)

When invoked with `--readme-only`, follow this process:

### Step 1: Scan Repository Structure

Use Glob and Read to gather:
```
- package.json, requirements.txt, go.mod, Cargo.toml (dependencies)
- README.md (existing, if any)
- src/, lib/, app/ structure
- Dockerfile, docker-compose.yml
- .github/workflows/, Jenkinsfile, .gitlab-ci.yml (CI/CD)
- terraform/, infrastructure/, deploy/ (IaC)
```

### Step 2: Detect Tech Stack

Build a tech stack table from detected files:

| Indicator File | Technology |
|----------------|------------|
| `package.json` | Node.js + listed dependencies |
| `requirements.txt` / `pyproject.toml` | Python + listed packages |
| `go.mod` | Go |
| `Cargo.toml` | Rust |
| `Dockerfile` | Docker |
| `*.tf` files | Terraform |
| `.github/workflows/` | GitHub Actions |

### Step 3: Generate README Sections

**Template:**

```markdown
# {Project Name}

> {One-line description - infer from existing README or package.json description}

## Overview

{AI-generated narrative: 2-3 paragraphs covering:
- What problem does this solve?
- What approach was taken?
- Key constraints or requirements}

## Tech Stack

| Layer | Technologies |
|-------|--------------|
| {layer} | {detected technologies} |

## Architecture

{If diagrams requested, embed mermaid. Otherwise:}
See [docs/architecture.md](docs/architecture.md) for system diagrams.

## Key Features

{Extract from existing README or infer from code structure:}
- Feature 1: Brief description
- Feature 2: Brief description

## My Contributions

{Placeholder for user to fill:}
- Designed and implemented [specific system/component]
- Led [initiative] resulting in [outcome]

## Code Highlights

{If safe code pack exists:}
See [/public](/public) for sanitized examples of:
- Infrastructure patterns
- API design
- Testing strategies

---

*Generated with shadow-clone on {date}*
```

### Step 4: Review Gate

Before writing output, display:
1. Preview of generated README
2. List of detected technologies
3. Any warnings (e.g., "No existing README found")

Ask user: **"Write this README to {output_path}? (yes/no)"**

### Step 5: Write Output

If approved, write to:
- `{output_dir}/README.md`
- Update `.shadow-clone-meta.json` with generation metadata

---

## Diagram Generation (`--diagrams-only`)

When invoked with `--diagrams-only`, generate C4 architecture diagrams in Mermaid format.

### Step 1: Scan Project Structure

Detect project characteristics:
```
- package.json, go.mod, requirements.txt, Cargo.toml (project type)
- docker-compose.yml, Dockerfile (container boundaries)
- .env, .env.example (external system hints)
- src/, apps/, packages/ (folder structure)
```

### Step 2: Detect External Systems

Scan for integrations with external services:

| Source | What to Look For |
|--------|------------------|
| `.env`, `.env.example` | `*_URL`, `*_API_KEY`, `*_HOST` patterns |
| Package dependencies | `pg`, `redis`, `stripe`, `aws-sdk`, `@prisma/client`, etc. |
| Config files | Database connection strings, third-party endpoints |
| HTTP clients | Base URLs in axios/fetch configurations |

**Common external system mappings:**

| Dependency | External System |
|------------|-----------------|
| `pg`, `mysql2`, `@prisma/client` | Database (PostgreSQL/MySQL) |
| `redis`, `ioredis` | Redis Cache |
| `stripe`, `@stripe/stripe-js` | Stripe Payment API |
| `aws-sdk`, `@aws-sdk/*` | AWS Services |
| `@sendgrid/mail`, `nodemailer` | Email Service |
| `elasticsearch`, `@elastic/*` | Elasticsearch |

### Step 3: Detect Containers

Identify logical deployment units:

| Source | Container Indicator |
|--------|---------------------|
| `docker-compose.yml` | Each `service:` = container |
| Folder structure | `src/api/`, `src/worker/`, `src/web/` |
| Monorepo config | `workspaces` in package.json, `apps/*` |
| Separate entry points | Multiple `main.ts`, `index.js` files |

**Fallback for monoliths:** If no container boundaries detected, treat as single container with rich component diagram.

### Step 4: Detect Components

Identify internal components within containers:

| Folder Pattern | Component Type |
|----------------|----------------|
| `controllers/`, `routes/`, `handlers/` | HTTP Handlers |
| `services/`, `domain/`, `core/` | Business Logic |
| `repositories/`, `data/`, `db/` | Data Access |
| `middleware/`, `interceptors/` | Middleware |
| `utils/`, `helpers/`, `lib/` | Utilities |
| `models/`, `entities/`, `schemas/` | Data Models |
| `events/`, `queues/`, `jobs/` | Async Processing |

### Step 5: Generate C4 Context Diagram

Create system context showing users and external systems:

```mermaid
C4Context
    title System Context diagram for {Project Name}

    Person(user, "User", "Primary system user")

    System(system, "{Project Name}", "{Description from package.json or inferred}")

    System_Ext(ext_db, "Database", "{Detected DB type}")
    System_Ext(ext_cache, "Cache", "Redis") %% if detected
    System_Ext(ext_payment, "Payment Provider", "Stripe") %% if detected

    Rel(user, system, "Uses")
    Rel(system, ext_db, "Reads/Writes data")
    Rel(system, ext_cache, "Caches data") %% if detected
```

### Step 6: Generate C4 Container Diagram

Show internal containers and their interactions:

```mermaid
C4Container
    title Container diagram for {Project Name}

    Person(user, "User", "Primary user")

    System_Boundary(boundary, "{Project Name}") {
        Container(api, "API Server", "{Tech stack}", "Handles HTTP requests")
        Container(worker, "Background Worker", "{Tech}", "Processes async jobs") %% if detected
        Container(web, "Web App", "{Framework}", "User interface") %% if detected
        ContainerDb(db, "Database", "{DB Type}", "Stores application data")
    }

    Rel(user, web, "Uses", "HTTPS")
    Rel(web, api, "Calls", "REST/GraphQL")
    Rel(api, db, "Reads/Writes")
    Rel(api, worker, "Enqueues jobs") %% if detected
```

### Step 7: Generate C4 Component Diagram

Detail components within the main container:

```mermaid
C4Component
    title Component diagram for {Container Name}

    Container_Boundary(api, "API Application") {
        Component(controllers, "Controllers", "{Framework}", "Handle HTTP requests, validate input")
        Component(services, "Services", "{Language}", "Business logic, orchestration")
        Component(repos, "Repositories", "{ORM/Driver}", "Data access, queries")
        Component(middleware, "Middleware", "{Framework}", "Auth, logging, error handling") %% if detected
    }

    Rel(controllers, services, "Uses")
    Rel(services, repos, "Uses")
    Rel(middleware, controllers, "Wraps")
```

### Step 8: Generate `docs/architecture.md`

Create prose documentation with embedded diagrams:

```markdown
# Architecture Overview

## System Description

{AI-generated narrative: 2-3 paragraphs describing:
- System purpose and scope
- Key architectural decisions inferred from stack
- Primary data flows}

## Tech Stack Summary

| Layer | Technology |
|-------|------------|
| Runtime | {Node.js/Python/Go/etc.} |
| Framework | {Express/FastAPI/Gin/etc.} |
| Database | {PostgreSQL/MongoDB/etc.} |
| Cache | {Redis/etc. if detected} |

## C4 Diagrams

### Level 1: System Context

{Embed context.mermaid}

### Level 2: Container

{Embed containers.mermaid}

### Level 3: Component

{Embed components.mermaid}

## Key Design Decisions

{Inferred from stack, e.g.:
- "Layered architecture with clear separation of concerns"
- "Event-driven processing for background jobs"
- "Repository pattern for data access abstraction"}
```

### Step 9: Review Gate

Before writing output, display:

1. **Detected Tech Stack:**
   ```
   Language: TypeScript
   Framework: Express.js
   Database: PostgreSQL (via Prisma)
   Cache: Redis (via ioredis)
   ```

2. **External Systems Found:**
   - Database: PostgreSQL
   - Cache: Redis
   - Payment: Stripe

3. **Containers Identified:**
   - API Server (src/api/)
   - Background Worker (src/worker/)

4. **Components Detected:**
   - Controllers (src/api/controllers/)
   - Services (src/api/services/)
   - Repositories (src/api/repositories/)

Ask user: **"Generate diagrams with this detected structure? (yes/no)"**

### Step 10: Write Output

If approved, write to:
- `{output_dir}/diagrams/context.mermaid`
- `{output_dir}/diagrams/containers.mermaid`
- `{output_dir}/diagrams/components.mermaid`
- `{output_dir}/docs/architecture.md`
- Update `.shadow-clone-meta.json` with generation metadata

### Edge Cases

| Scenario | Handling |
|----------|----------|
| No docker-compose | Use folder structure for container detection |
| Monolith (no boundaries) | Single container, rich component diagram |
| Unknown technology | Use generic labels, note uncertainty in review |
| No components detected | Generate context + container only, skip component |
| Multiple services | Generate component diagram for each major service |

---

## Secret Scanning

Secret scanning detects API keys, credentials, tokens, and other sensitive patterns in code, producing a sensitivity map for Stage 3 sanitization.

### Step 1: Define Detection Patterns

Use regex-based pattern matching for common secret formats:

| Type | Regex Pattern | Example Match |
|------|---------------|---------------|
| AWS Access Key | `AKIA[0-9A-Z]{16}` | `AKIAIOSFODNN7EXAMPLE` |
| AWS Secret Key | `(?i)aws.{0,20}secret.{0,20}['\"][0-9a-zA-Z/+]{40}['\"]` | - |
| GitHub PAT | `ghp_[a-zA-Z0-9]{36}` | `ghp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx` |
| GitHub OAuth | `gho_[a-zA-Z0-9]{36}` | - |
| Stripe Live | `sk_live_[a-zA-Z0-9]{24}` | - |
| Stripe Test | `sk_test_[a-zA-Z0-9]{24}` | - |
| Generic API Key | `(?i)(api[_-]?key\|apikey)['\"]?\s*[:=]\s*['\"][a-zA-Z0-9]{16,}['\"]` | - |
| Database URL | `(?i)(postgres\|mysql\|mongodb\|redis)://[^\s'"]+` | `postgres://user:pass@host/db` |
| Private Key | `-----BEGIN [A-Z ]+ PRIVATE KEY-----` | - |
| JWT Token | `eyJ[a-zA-Z0-9\-_]+\.[a-zA-Z0-9\-_]+\.` | - |
| Bearer Token | `(?i)bearer\s+[a-zA-Z0-9\-_.]+` | - |
| Slack Token | `xox[baprs]-[a-zA-Z0-9-]+` | - |
| Google API | `AIza[0-9A-Za-z\-_]{35}` | - |
| Internal IP | `(192\.168\.\|10\.\|172\.(1[6-9]\|2[0-9]\|3[01])\.)[0-9.]+` | `192.168.1.100` |
| Internal URL | `https?://[a-z0-9.-]+\.(internal\|corp\|local\|dev)[:/]` | - |

### Step 2: Identify Files to Scan

**Priority targets:**
```
- .env, .env.*, *.env (environment files)
- config/, settings/ (configuration directories)
- *.json, *.yml, *.yaml (config formats)
- src/**/*.ts, src/**/*.js (source code)
- docker-compose.yml, Dockerfile
- *.pem, *.key, *.p12 (certificate files - flag entire file)
```

**Skip by default:**
```
- node_modules/, vendor/, .git/
- *.lock files
- Binary files
- Files > 1MB
```

### Step 3: Scan and Classify

For each file:
1. Read content line-by-line
2. Match against pattern library
3. Classify severity:

| Severity | Criteria |
|----------|----------|
| CRITICAL | Private keys, active database credentials, cloud provider secrets |
| HIGH | API keys, OAuth tokens, JWT secrets |
| MEDIUM | Internal URLs, hardcoded passwords in comments |
| LOW | Email patterns, potential PII, placeholder-looking secrets |

### Step 4: Build Sensitivity Map

Generate JSON structure:

```json
{
  "scan_timestamp": "ISO-8601",
  "files_scanned": 245,
  "secrets_found": 12,
  "by_severity": { "critical": 2, "high": 5, "medium": 3, "low": 2 },
  "by_type": { "api_keys": 4, "database_credentials": 3 },
  "findings": [
    {
      "file": "src/config/db.ts",
      "line": 15,
      "type": "database_credentials",
      "severity": "critical",
      "match": "postgres://***:***@..."
    }
  ]
}
```

### Step 5: Generate Review Report

Display format:

```
SECRET SCAN RESULTS
═══════════════════

Summary: 12 secrets found in 8 files

By Severity:
  CRITICAL  2 files  ⛔ Must exclude or redact
  HIGH      5 files  ⚠️  Review for redaction
  MEDIUM    3 files
  LOW       2 files

Critical Findings:
  src/config/db.ts:15 - database_credentials
  .env:3 - aws_secret_key

Recommendations:
  1. EXCLUDE: src/config/db.ts (active credentials)
  2. REDACT: .env (replace with placeholders)
```

### Step 6: Review Gate

Before proceeding to sanitization:
1. Display scan summary
2. Show critical/high severity files
3. List recommended actions
4. Ask: **"Proceed with these findings to sanitization? (yes/no)"**

### Step 7: Output

Write sensitivity map to:
- In-memory for Stage 3 consumption
- Include in `.shadow-clone-meta.json` under `secret_scan` key

### Edge Cases

| Scenario | Handling |
|----------|----------|
| `.env.example` with placeholders | Flag as LOW, suggest keeping as documentation |
| Test fixtures with fake secrets | Detect `test`, `example`, `sample` in path → reduce severity |
| Large binary files | Skip scanning, log as skipped |
| Minified JS | Scan but note reduced accuracy |
| Base64-encoded secrets | Decode and scan common patterns |

---

## Implementation Status

| Component | Status |
|-----------|--------|
| Argument parsing | Basic |
| Secret scanning | **Implemented** |
| README generation | **Implemented** |
| Diagram generation | **Implemented** |
| Safe code pack | Not started |
| Review gate | Basic |
