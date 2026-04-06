---
name: repo-analyzer
description: Analyze any codebase and generate LLM-friendly summaries, dependency graphs, and structured documentation. Inspired by GitVizz. Scans repo structure, parses AST for Python/JS/TS, maps imports and function calls, generates context documents. Use when onboarding to a new codebase, reviewing architecture, or building context for complex tasks.
---

# Repo Analyzer

Analyze any local repository and produce structured, LLM-friendly context. Absorbed from GitVizz's core analysis patterns.

## What It Does

1. **Structure Map** — File tree with sizes, languages, entry points
2. **Dependency Graph** — Who imports who, function call chains
3. **Context Document** — LLM-optimized summary of the entire codebase
4. **Documentation** — Auto-generated docs per module/component

## Usage

### Quick Analysis
```
"Analyze repo: /path/to/project"
"What does this codebase do: ~/council-room"
```

### Deep Analysis
```
"Deep analyze: /path/to/project"
```

## Agent Procedure

### Step 1: Structure Scan
```bash
# Get file tree with stats
find <repo> -type f \
  -not -path "*/node_modules/*" \
  -not -path "*/.git/*" \
  -not -path "*/dist/*" \
  -not -path "*/__pycache__/*" \
  -not -path "*/build/*" \
  | head -500

# Count by language
find <repo> -type f -name "*.py" | wc -l    # Python
find <repo> -type f -name "*.ts" -o -name "*.tsx" | wc -l  # TypeScript
find <repo> -type f -name "*.js" -o -name "*.jsx" | wc -l  # JavaScript
find <repo> -type f -name "*.rs" | wc -l    # Rust
find <repo> -type f -name "*.go" | wc -l    # Go

# Total lines of code
find <repo> -type f \( -name "*.py" -o -name "*.ts" -o -name "*.js" -o -name "*.tsx" -o -name "*.jsx" \) \
  -not -path "*/node_modules/*" | xargs wc -l 2>/dev/null | tail -1

# Find entry points
ls <repo>/package.json <repo>/main.* <repo>/index.* <repo>/server.* <repo>/app.* 2>/dev/null
cat <repo>/package.json 2>/dev/null | jq '.main, .scripts.start, .scripts.dev' 2>/dev/null
```

### Step 2: Dependency Mapping
```bash
# JavaScript/TypeScript imports
grep -rn "^import\|^const.*require" <repo>/src/ --include="*.ts" --include="*.js" --include="*.tsx" | head -100

# Python imports  
grep -rn "^import\|^from.*import" <repo>/ --include="*.py" | head -100

# package.json dependencies
cat <repo>/package.json 2>/dev/null | jq '{deps: .dependencies, devDeps: .devDependencies}' 2>/dev/null

# Python requirements
cat <repo>/requirements.txt <repo>/pyproject.toml 2>/dev/null | head -50
```

### Step 3: Key File Analysis
Read the most important files (entry points, configs, core modules):
```bash
# Find largest source files (usually the most important)
find <repo>/src -type f \( -name "*.ts" -o -name "*.py" -o -name "*.js" \) \
  -not -path "*/node_modules/*" -exec wc -l {} \; | sort -rn | head -20

# Read entry point
cat <repo>/src/main.* <repo>/src/index.* <repo>/src/app.* 2>/dev/null | head -100

# Read config
cat <repo>/.env.example <repo>/config.* 2>/dev/null | head -50
```

### Step 4: Generate Context Document

After scanning, produce a structured markdown document:

```markdown
# Repository Analysis: {repo_name}

## Overview
- **Purpose:** {one-line description}
- **Language:** {primary language}
- **Framework:** {framework if detected}
- **Size:** {files} files, {lines} lines of code

## Architecture
{high-level description of how components connect}

## Key Components
| Component | Path | Purpose | Lines |
|-----------|------|---------|-------|
| {name} | {path} | {what it does} | {loc} |

## Dependencies
### External
{list of external packages with versions}

### Internal
{how modules depend on each other}

## Entry Points
{how the app starts, main execution flow}

## Configuration
{env vars, config files, what needs to be set}

## Patterns Observed
{coding patterns, architecture patterns, notable decisions}
```

### Step 5: Save Output

Save to workspace or Obsidian vault:
```
~/.openclaw/workspace/analysis/{repo-name}-analysis.md
```

Or for Obsidian:
```
~/livesync-bridge/vault/References/{repo-name}-analysis.md
```

## Dependency Graph (Text-Based)

Since we can't render visual graphs in terminal/WhatsApp, use text-based dependency graphs:

```
Main Entry (server.js)
├── routes/
│   ├── api.js → controllers/userController.js
│   ├── auth.js → services/authService.js
│   └── health.js (standalone)
├── services/
│   ├── authService.js → models/User.js, utils/jwt.js
│   └── emailService.js → utils/templates.js
├── models/
│   ├── User.js → mongoose
│   └── Session.js → mongoose
└── utils/
    ├── jwt.js → jsonwebtoken
    └── templates.js (standalone)
```

For visual graphs, generate HTML using the visual-explainer skill.

## Integration

### With Autoskills
Run autoskills detection first, then deep analyze:
```
1. bash skills/autoskills/scripts/detect.sh /path/to/repo
2. "Analyze repo: /path/to/repo" (this skill)
3. Combined output = full onboarding document
```

### With Pipeline Orchestrator
Before starting work on a new codebase:
```
1. Analyze repo (this skill)
2. ULTRAPLAN based on analysis
3. Pipeline execute the plan
```

### With Secret Scanner
After analysis, scan for leaked secrets:
```
bash skills/secret-scanner/scripts/scan.sh /path/to/repo
```

## Limitations

- AST parsing is bash-based (grep/read), not full tree-sitter like GitVizz
- Dependency graph is text-based, not interactive
- Works best with JS/TS/Python. Other languages get basic structure only.
- Large repos (>10K files) should use --quick mode (structure only, skip deep analysis)
