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
| `--diagram-format FORMAT` | `mermaid` (default) or `drawio` for .drawio.svg files |

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

## Execution Instructions

When this skill is invoked, follow these steps in order.

### Parse Arguments

Extract from invocation:
1. **Target path** - First positional argument (required)
2. **Mode** - `--readme-only`, `--diagrams-only`, `--safe-code-only`, or full (default)
3. **Rules** - `--rules "natural language rules"`
4. **Output** - `--output DIR` (default: `./shadow-clone-output/`)
5. **Safety** - `--mode conservative` (default) or `--mode moderate`
6. **Diagram format** - `--diagram-format mermaid` (default) or `--diagram-format drawio`

**Validation:**
1. Use `Bash: ls {target_path}` to verify path exists
2. If path doesn't exist, stop and report error
3. Create output directory: `Bash: mkdir -p {output_dir}`

### Execute Pipeline

Based on mode flag, execute stages:

| Mode | Stages to Run |
|------|---------------|
| `--readme-only` | 1 → 2 → 4a → 5 |
| `--diagrams-only` | 1 → 2 → 4b → 5 |
| `--safe-code-only` | 1 → 2 → 3 → 4c → 5 |
| Full (default) | 1 → 2 → 3 → 4 → 5 |

**Note:** When `--diagrams-only` or full mode runs Stage 4b, pass the `--diagram-format` value to determine output format (`.mermaid` or `.drawio.svg`).

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

**Execution:**

**Step 1.1: Scan project structure**

Use Glob to find key files:
- `Glob: **/package.json` - Node.js projects
- `Glob: **/requirements.txt` OR `**/pyproject.toml` - Python projects
- `Glob: **/go.mod` - Go projects
- `Glob: **/Cargo.toml` - Rust projects
- `Glob: **/docker-compose.yml` - Container config
- `Glob: **/.github/workflows/*.yml` - CI/CD

**Step 1.2: Read project manifests**

Read the first matching manifest to detect:
- Project name and description
- Dependencies (tech stack)
- Scripts/commands available

**Step 1.3: Build file inventory**

Use Glob to count files by type:
- `Glob: **/*.ts` OR `**/*.js` - TypeScript/JavaScript
- `Glob: **/*.py` - Python
- `Glob: **/*.go` - Go

**Skip by default (never scan):**
- `node_modules/`, `vendor/`, `.git/`, `__pycache__/`, `dist/`, `build/`
- `*.lock` files
- Binary files (`.png`, `.jpg`, `.exe`, `.dll`, etc.)

### Stage 2: Analyze
- **Secret scanning** - API keys, credentials, tokens
- **IP detection** - Proprietary algorithms, business logic
- **PII scanning** - Names, emails, internal URLs
- Build sensitivity map per file

**Execution:**

**Step 2.1: Scan for secrets**

Run Grep for each pattern category. Use `output_mode: "content"` to see matches:

| Pattern | Grep Command |
|---------|--------------|
| AWS Access Key | `Grep: pattern="AKIA[0-9A-Z]{16}"` |
| AWS Secret | `Grep: pattern="(?i)aws.{0,20}secret.{0,20}['\"][0-9a-zA-Z/+]{40}"` |
| GitHub PAT | `Grep: pattern="ghp_[a-zA-Z0-9]{36}"` |
| Stripe Key | `Grep: pattern="sk_(live\|test)_[a-zA-Z0-9]{24}"` |
| Database URL | `Grep: pattern="(?i)(postgres\|mysql\|mongodb\|redis)://[^\s'\"]+", -i=true` |
| Private Key | `Grep: pattern="-----BEGIN .* PRIVATE KEY-----"` |
| JWT Token | `Grep: pattern="eyJ[a-zA-Z0-9\-_]+\.[a-zA-Z0-9\-_]+\."` |
| Generic API Key | `Grep: pattern="(?i)(api[_-]?key)['\"]?\s*[:=]\s*['\"][a-zA-Z0-9]{16,}"` |
| Internal URL | `Grep: pattern="https?://[a-z0-9.-]+\.(internal\|corp\|local)[:/]"` |

**Step 2.2: Classify findings**

For each match, assign severity:
- **CRITICAL**: Private keys, database credentials with passwords, cloud provider secrets
- **HIGH**: API keys, OAuth tokens, JWT secrets
- **MEDIUM**: Internal URLs, hardcoded passwords in comments
- **LOW**: Placeholder-looking values, test fixtures

**Reduce severity by one level if:**
- Path contains `test/`, `spec/`, `example/`, `fixture/`
- File is `.env.example` or `.env.sample`
- Value contains `example`, `sample`, `test`, `xxx`, `placeholder`

**Step 2.3: Build sensitivity map**

Track findings mentally for Stage 3:
- Files with CRITICAL findings → exclude entirely
- Files with HIGH findings → exclude or redact (based on mode)
- Files with MEDIUM/LOW → redact inline

### Stage 3: Sanitize
- Apply allow/deny rules
- Redact sensitive content
- Generate placeholder replacements

### Stage 4: Generate
Based on mode flags, generate:
- **README** - Project overview, tech stack, architecture summary
- **Diagrams** - Mermaid C4 diagrams (Context, Container, Component levels)
- **Safe Code Pack** - Allowlisted files with sanitization applied
- **Decision Logs** - Architecture Decision Records (ADRs) extracted from commits/code

### Stage 5: Review Gate
- Display summary of what will be published
- Show any warnings or flagged content
- **Require explicit user approval before output**

**Execution:**

**CRITICAL: Never write output without explicit user approval.**

Before writing any files, display this summary:

```
══════════════════════════════════════════════════════════
SHADOW-CLONE REVIEW
══════════════════════════════════════════════════════════

Target: {target_path}
Output: {output_dir}
Mode: {mode}

WILL GENERATE:
  {list of files to be created}

SECRETS FOUND:
  CRITICAL: {n} (files will be EXCLUDED)
  HIGH: {n} (will be {excluded|redacted} based on mode)
  MEDIUM: {n} (will be redacted)
  LOW: {n} (will be redacted)

{If --safe-code-only or full mode:}
SAFE CODE PACK:
  Include: {n} files
  Exclude: {n} files
  Redact: {n} files

══════════════════════════════════════════════════════════
```

**Ask user:** "Proceed with generation? (yes/no)"

- If **yes**: Execute Stage 4 output writing
- If **no**: Ask what to adjust and re-run relevant stages

## Output Structure

**Default (Mermaid format):**
```
shadow-clone-output/
├── README.md                    # Main project narrative
├── REVIEW.md                    # What's included/excluded, flags for review
├── .shadow-clone-meta.json      # Provenance: source, config, timestamp
├── docs/
│   ├── architecture.md          # C4 diagrams with explanations
│   ├── decisions/               # Architecture Decision Records (ADRs)
│   │   ├── INDEX.md
│   │   └── 001-*.md
│   └── flows/
├── diagrams/                    # Standalone diagram files
│   ├── context.mermaid
│   ├── containers.mermaid
│   └── components.mermaid
└── code/                        # Safe code pack
    ├── infrastructure/
    ├── utils/
    └── stubs/                   # Stubbed excluded code (if enabled)
```

**With `--diagram-format drawio`:**
```
shadow-clone-output/
├── ...                          # Same as above
├── diagrams/                    # Draw.io SVG files (GitHub-native, editable)
│   ├── context.drawio.svg
│   ├── containers.drawio.svg
│   └── components.drawio.svg
└── ...
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

## Technical Trade-offs

{Auto-generate from detected decisions. Show what was optimized for AND what wasn't:}

| Decision | Optimized For | Trade-off Accepted |
|----------|---------------|-------------------|
| {e.g., PostgreSQL} | {Data integrity, complex queries} | {Operational complexity vs SQLite} |
| {e.g., Redis Queue} | {Reliability, burst handling} | {Eventual consistency vs sync} |
| {e.g., Monolith} | {Simplicity, fast iteration} | {Scaling limits vs microservices} |

{Link to detailed decisions:}
See [docs/decisions/](docs/decisions/) for Architecture Decision Records.

## My Contributions

{Placeholder for user to fill:}
- Designed and implemented [specific system/component]
- Led [initiative] resulting in [outcome]

## Code Highlights

{If safe code pack exists:}
See [/code](/code) for sanitized examples of:
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

## Draw.io SVG Format (`--diagram-format drawio`)

When `--diagram-format drawio` is specified, generate `.drawio.svg` files instead of Mermaid. These files render natively on GitHub as images while remaining fully editable in draw.io.

### How .drawio.svg Works

A `.drawio.svg` file is a valid SVG image with embedded draw.io metadata:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg PUBLIC "-//W3C//DTD SVG 1.1//EN" "http://www.w3.org/Graphics/SVG/1.1/DTD/svg11.dtd">
<svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink"
     version="1.1" width="{width}px" height="{height}px" viewBox="0 0 {width} {height}">
  <defs>
    <!-- draw.io metadata embedded here -->
    <mxfile host="shadow-clone" modified="{timestamp}" agent="shadow-clone" version="1.0">
      <diagram id="{diagram-id}" name="{Diagram Name}">
        <mxGraphModel dx="0" dy="0" grid="1" gridSize="10" guides="1" tooltips="1"
                      connect="1" arrows="1" fold="1" page="1" pageScale="1">
          <root>
            <mxCell id="0"/>
            <mxCell id="1" parent="0"/>
            <!-- Diagram elements as mxCell nodes -->
          </root>
        </mxGraphModel>
      </diagram>
    </mxfile>
  </defs>
  <!-- SVG visual content -->
  <g>
    <!-- Rectangles, text, arrows rendered as SVG -->
  </g>
</svg>
```

### C4 Styling Rules

Apply consistent C4 styling for professional appearance:

| Element | Style |
|---------|-------|
| Person | Rounded rectangle, fill `#438DD5`, white text |
| System | Rectangle, fill `#1168BD`, white text |
| External System | Rectangle, fill `#999999`, white text |
| Container | Rectangle, fill `#438DD5`, white text |
| Container (DB) | Cylinder shape, fill `#438DD5`, white text |
| Component | Rectangle, fill `#85BBF0`, black text |
| System Boundary | Dashed rectangle, no fill, `#888888` stroke |
| Relationships | Solid arrow, `#707070`, label in gray |

**Design principles:**
- Minimal aesthetic - simple shapes, no gradients or shadows
- Monochrome-friendly - works in light and dark themes
- Clear typography - consistent font sizing
- Adequate spacing - 20px minimum between elements

### mxCell Format Reference

**Rectangle (System/Container/Component):**
```xml
<mxCell id="{id}" value="{label}" style="rounded=0;whiteSpace=wrap;html=1;fillColor=#1168BD;fontColor=#ffffff;strokeColor=#0B4884;"
        vertex="1" parent="1">
  <mxGeometry x="{x}" y="{y}" width="120" height="60" as="geometry"/>
</mxCell>
```

**Person:**
```xml
<mxCell id="{id}" value="{label}" style="shape=umlActor;verticalLabelPosition=bottom;verticalAlign=top;html=1;fillColor=#438DD5;strokeColor=#3A7BBD;"
        vertex="1" parent="1">
  <mxGeometry x="{x}" y="{y}" width="30" height="60" as="geometry"/>
</mxCell>
```

**Database (Cylinder):**
```xml
<mxCell id="{id}" value="{label}" style="shape=cylinder3;whiteSpace=wrap;html=1;boundedLbl=1;backgroundOutline=1;size=15;fillColor=#438DD5;fontColor=#ffffff;strokeColor=#3A7BBD;"
        vertex="1" parent="1">
  <mxGeometry x="{x}" y="{y}" width="60" height="80" as="geometry"/>
</mxCell>
```

**Boundary (Dashed box):**
```xml
<mxCell id="{id}" value="{label}" style="rounded=0;whiteSpace=wrap;html=1;fillColor=none;dashed=1;strokeColor=#888888;verticalAlign=top;fontStyle=1;"
        vertex="1" parent="1">
  <mxGeometry x="{x}" y="{y}" width="400" height="300" as="geometry"/>
</mxCell>
```

**Relationship (Arrow):**
```xml
<mxCell id="{id}" value="{label}" style="edgeStyle=orthogonalEdgeStyle;rounded=0;orthogonalLoop=1;jettySize=auto;html=1;strokeColor=#707070;fontColor=#707070;"
        edge="1" parent="1" source="{source-id}" target="{target-id}">
  <mxGeometry relative="1" as="geometry"/>
</mxCell>
```

### Generating Context Diagram (drawio)

When `--diagram-format drawio`, Step 5 produces `context.drawio.svg`:

1. Create SVG with mxfile metadata in `<defs>`
2. Add Person mxCell for user at top center
3. Add System mxCell for main system in center
4. Add System_Ext mxCells for each external system (DB, cache, etc.)
5. Add edge mxCells for each relationship
6. Render corresponding SVG shapes in `<g>` element
7. Calculate viewBox from element positions

### Generating Container Diagram (drawio)

When `--diagram-format drawio`, Step 6 produces `containers.drawio.svg`:

1. Create boundary mxCell encompassing all internal containers
2. Add Container mxCells inside boundary
3. Add ContainerDb (cylinder) mxCells for databases
4. Add external Person and System_Ext outside boundary
5. Add relationship edges between elements

### Generating Component Diagram (drawio)

When `--diagram-format drawio`, Step 7 produces `components.drawio.svg`:

1. Create Container_Boundary mxCell for the container being detailed
2. Add Component mxCells for each detected component
3. Use lighter fill (`#85BBF0`) to distinguish from containers
4. Add inter-component relationship edges

### Compatibility Matrix

| Tool | Support |
|------|---------|
| GitHub rendering | ✅ Displays as image in markdown |
| draw.io desktop | ✅ Full editing capability |
| diagrams.net (web) | ✅ Full editing capability |
| VS Code Draw.io Extension | ✅ Full editing capability |
| Browser (direct) | ✅ Displays as SVG image |
| Figma/Sketch import | ✅ Imports as vector graphic |

### Output Files (drawio format)

When `--diagram-format drawio`, Step 10 writes:
- `{output_dir}/diagrams/context.drawio.svg`
- `{output_dir}/diagrams/containers.drawio.svg`
- `{output_dir}/diagrams/components.drawio.svg`
- `{output_dir}/docs/architecture.md` (with `![](../diagrams/*.drawio.svg)` embeds)

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

## Safe Code Pack (`--safe-code-only`)

Extract safe, shareable code by filtering through allow/deny rules and sanitizing detected secrets.

### Step 1: Define Default Allowlists

Categories of typically-safe code:

| Category | Patterns | Rationale |
|----------|----------|-----------|
| Infrastructure | `infrastructure/`, `deploy/`, `terraform/`, `k8s/` | Shows DevOps skills, rarely contains secrets |
| Utilities | `utils/`, `helpers/`, `lib/`, `common/` | Generic, reusable code |
| Types/Interfaces | `types/`, `interfaces/`, `*.d.ts` | Contract definitions, no logic |
| Tests | `__tests__/`, `*.test.*`, `*.spec.*` | Demonstrates testing practices |
| Config examples | `*.example`, `*.sample`, `*.template` | Documentation, placeholder values |

Default denylists:

| Category | Patterns | Rationale |
|----------|----------|-----------|
| Environment | `.env`, `.env.*` (not .example) | Contains real secrets |
| Credentials | `credentials/`, `secrets/`, `*.pem`, `*.key` | Sensitive by nature |
| Business logic | `billing/`, `pricing/`, `auth/` | Proprietary IP |
| Database | `migrations/`, `seeds/` | May contain real data patterns |

### Step 2: Apply User Rules

Parse `--rules` flag or config file:

```
# Natural language examples:
--rules "include infrastructure, exclude billing"
--rules "only utils and types"
--rules "everything except auth and payments"
```

Rule precedence:
1. Explicit deny (highest priority)
2. Explicit allow
3. Default denylist
4. Default allowlist
5. Deny by default (lowest - everything else excluded)

### Step 3: Filter Files

For each file in repository:
1. Check against denylists first (skip if matched)
2. Check against allowlists (include if matched)
3. Check against user rules
4. Apply sensitivity map:
   - CRITICAL severity → exclude entirely
   - HIGH severity → exclude or redact (user choice)
   - MEDIUM/LOW → redact inline

Output: List of files to include with redaction requirements

### Step 4: Sanitize Content

For each included file with secrets:
1. Read file content
2. For each finding from sensitivity map:
   - Replace secret value with placeholder
   - Preserve surrounding code structure

Placeholder format:
```
# Original:
const API_KEY = "sk_live_abc123xyz789";
const DB_URL = "postgres://user:pass@host/db";

# Sanitized:
const API_KEY = "REDACTED_STRIPE_KEY";
const DB_URL = "postgres://REDACTED_USER:REDACTED_PASS@REDACTED_HOST/db";
```

Placeholder naming:

| Secret Type | Placeholder |
|-------------|-------------|
| AWS Key | `REDACTED_AWS_KEY` |
| Database URL | `postgres://REDACTED/...` |
| API Key | `REDACTED_API_KEY` |
| Private Key | `[PRIVATE KEY REDACTED]` |
| JWT | `REDACTED_JWT_TOKEN` |
| Generic | `REDACTED_SECRET` |

### Step 5: Generate Output

Write sanitized files to output:
```
shadow-clone-output/
└── code/
    ├── infrastructure/
    │   └── docker-compose.yml
    ├── utils/
    │   ├── logger.ts
    │   └── validation.ts
    └── MANIFEST.md
```

Generate `MANIFEST.md`:
```markdown
# Safe Code Pack Manifest

Generated: {timestamp}
Source: {repo_path}

## Included Files

| File | Status | Redactions |
|------|--------|------------|
| infrastructure/docker-compose.yml | Sanitized | 2 secrets redacted |
| utils/logger.ts | Clean | - |
| utils/validation.ts | Clean | - |

## Excluded Files

| Category | Count | Reason |
|----------|-------|--------|
| Business logic | 12 | Default denylist |
| Contains secrets | 3 | CRITICAL severity |
| User rules | 5 | Excluded by --rules |

## Redaction Summary

| Type | Count |
|------|-------|
| Database URLs | 1 |
| API Keys | 1 |
| Total | 2 |
```

### Step 6: Review Gate

Display before writing:

```
SAFE CODE PACK PREVIEW
══════════════════════

Files to include: 15 files (3 with redactions)

By Category:
  infrastructure/    4 files
  utils/             6 files
  types/             5 files

Redactions:
  infrastructure/docker-compose.yml
    Line 12: DATABASE_URL → REDACTED
    Line 15: REDIS_URL → REDACTED

Excluded (45 files):
  - src/billing/ (12 files) - business logic
  - src/auth/ (8 files) - business logic
  - .env (1 file) - CRITICAL secrets
  - src/config/db.ts (1 file) - CRITICAL secrets
```

Ask: **"Generate safe code pack with these settings? (yes/no)"**

### Step 7: Write Output

If approved:
- Copy sanitized files to `{output_dir}/code/`
- Write `MANIFEST.md`
- Update `.shadow-clone-meta.json`:

```json
{
  "safe_code_pack": {
    "generated_at": "ISO-8601",
    "files_included": 15,
    "files_excluded": 45,
    "redactions_applied": 2,
    "rules_applied": ["include infrastructure", "exclude billing"]
  }
}
```

### Edge Cases

| Scenario | Handling |
|----------|----------|
| Zero files after filtering | Warn user, suggest relaxing rules |
| File with only secrets | Exclude entirely, note in manifest |
| Binary files | Skip, note as "binary excluded" |
| Symlinks | Resolve and copy if target is allowed |
| Empty directories | Don't create in output |
| Very large files (>1MB) | Include but warn in review |

---

## Decision Log / ADR Generation

Generate Architecture Decision Records (ADRs) to document key technical decisions.

### Step 1: Extract Decision Signals

Scan for decision indicators in:

**Commit messages:**
- Keywords: `decided`, `chose`, `vs`, `instead of`, `switched to`, `migrated`
- Pattern: `Grep: pattern="(decided|chose|vs\.|instead of|switched to|migrated)"` in `.git/logs`

**Code comments:**
- Keywords: `TODO: reconsider`, `DECISION:`, `NOTE: we chose`, `tradeoff`
- Pattern: `Grep: pattern="(TODO.*reconsider|DECISION:|NOTE:.*chose|tradeoff|trade-off)"`

**Configuration choices:**
- Database selection (PostgreSQL vs MySQL vs MongoDB)
- Framework selection (Express vs Fastify vs Nest)
- State management (Redux vs Context vs Zustand)

### Step 2: Identify Common Decision Categories

| Category | Detection Pattern |
|----------|-------------------|
| Database | `postgres`, `mysql`, `mongodb`, `redis`, `prisma` in dependencies |
| Queue/Messaging | `bull`, `rabbitmq`, `kafka`, `sqs` in dependencies |
| Auth Strategy | `passport`, `jwt`, `oauth`, `auth0` in dependencies |
| API Style | `graphql`, `rest`, `grpc`, `trpc` in code patterns |
| Testing Strategy | `jest`, `mocha`, `pytest`, `vitest` in dependencies |
| Deployment | `docker`, `kubernetes`, `serverless`, `vercel` in configs |

### Step 3: Generate ADR Template

For each detected decision, generate `docs/decisions/NNN-decision-name.md`:

```markdown
# Decision: {Title}

**Status:** Accepted
**Date:** {Inferred from commits or "Unknown"}
**Context:** {Auto-generated from detected patterns}

## Decision

{What was chosen - detected from dependencies/config}

## Rationale

{Placeholder for user to fill}
- Why this approach?
- What alternatives were considered?

## Consequences

**Benefits:**
- {Inferred benefit based on choice}

**Trade-offs:**
- {Inferred trade-off based on choice}

## Alternatives Considered

- {Alternative 1}: {Why not chosen}
- {Alternative 2}: {Why not chosen}

---
*Generated by shadow-clone. Please customize with project-specific context.*
```

### Step 4: Common Decision Templates

Pre-populate based on detected patterns:

**Database Choice (PostgreSQL detected):**
```markdown
# Decision: Use PostgreSQL as Primary Database

**Status:** Accepted

## Decision
PostgreSQL for relational data storage with Prisma ORM.

## Rationale
- ACID compliance for transactional integrity
- Rich query capabilities (JSON, full-text search)
- Strong ecosystem and tooling

## Trade-offs
- More operational complexity than SQLite
- Requires connection pooling at scale
- Schema migrations needed for changes
```

**Queue System (Bull/Redis detected):**
```markdown
# Decision: Use Queue for Async Processing

**Status:** Accepted

## Decision
Redis-backed job queue (Bull) for background processing.

## Rationale
- Decouple long-running tasks from request/response
- Burst handling and rate limiting
- Retry logic with dead letter queue

## Trade-offs
- Eventual consistency (not immediate)
- Additional infrastructure (Redis)
- Complexity in job monitoring/debugging
```

### Step 5: Output Structure

```
shadow-clone-output/
└── docs/
    └── decisions/
        ├── 001-database-choice.md
        ├── 002-authentication-strategy.md
        ├── 003-api-architecture.md
        └── INDEX.md
```

Generate `docs/decisions/INDEX.md`:
```markdown
# Architecture Decisions

| # | Decision | Status | Date |
|---|----------|--------|------|
| 001 | Database Choice | Accepted | Inferred |
| 002 | Authentication Strategy | Accepted | Inferred |
| 003 | API Architecture | Accepted | Inferred |

---
*Decisions extracted by shadow-clone. Customize with your rationale.*
```

### Step 6: Review Gate

Display decision summary before generation:

```
DECISIONS DETECTED:
  Database: PostgreSQL (via Prisma)
  Auth: JWT-based (via jsonwebtoken)
  Queue: Redis + Bull
  API: REST (Express routes detected)

ADRs to generate: 4 files in docs/decisions/
```

---

## Implementation Status

| Component | Status |
|-----------|--------|
| Argument parsing | **Implemented** |
| Stage 1: Ingest | **Implemented** |
| Stage 2: Analyze | **Implemented** |
| Stage 3: Sanitize | **Implemented** |
| Stage 4: Generate | **Implemented** |
| Stage 5: Review gate | **Implemented** |
