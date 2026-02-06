---
description: Quick-start guide for the AI Research Agent system
---

# AI Research Agent — Setup Guide

This system lets you ask plain-English questions about any codebase and get comprehensive answers. It works by generating structured documentation, building index maps, and using sub-agents to efficiently discover and synthesize information.

## Prerequisites

The command files are installed globally at `~/.claude/commands/` and are available in every project. No per-project installation needed.

## Getting Started

### 1. Navigate to your project
```
cd /path/to/your/project
```

### 2. Generate documentation
```
/generate-docs
```
This scans your codebase and creates structured YAML documentation in `documentation/`:
- `openapi/` — API endpoint specs per service
- `internalfunctions/` — Function documentation per service
- `events/` — Event/listener documentation per service

It also creates `ai-config.json` if it doesn't exist.

### 3. Generate index maps
```
/generate-maps
```
This creates two JSON index files:
- `ai-docs-map.json` — Maps service names and keywords to documentation files
- `ai-service-map.json` — Maps routes, actions, and source files for fast lookup

### 4. Ask questions
```
/research "How does authentication work?"
/research "What happens when a user places an order?"
/research "How does the Z Society tab system work?"
```

The research agent will:
1. Spawn sub-agents to find relevant docs and source files
2. Read the identified files
3. Follow cross-references iteratively (up to 3 rounds)
4. Synthesize a comprehensive answer
5. Write the report to `documentation/Research.md`

## Updating Documentation

After making code changes:

```
# Update docs for a specific service
/generate-docs Auth

# Regenerate all docs
/generate-docs

# Rebuild index maps
/generate-maps
```

## File Structure

```
your-project/
├── ai-config.json               # Project config (auto-created)
├── ai-docs-map.json             # Documentation index (auto-generated)
├── ai-service-map.json          # Source code index (auto-generated)
└── documentation/
    ├── openapi/
    │   └── [ServiceName].yml
    ├── internalfunctions/
    │   └── [ServiceName].yml
    ├── events/
    │   └── [ServiceName].yml
    └── Research.md              # Latest research report
```

## Tips

- **Be specific with questions.** "How does auth work?" is good. "How does the login flow validate credentials and create a session?" is better.
- **Regenerate docs after major changes.** The research agent's quality depends on up-to-date documentation and maps.
- **Use service-specific regeneration.** `/generate-docs Auth` is faster than regenerating everything when you only changed auth code.
- **Check Research.md.** The full research report is saved there for reference, including the discovery path showing how the answer was found.

## Architecture

```
/research "question"
    │
    ├─► Documentation Locator (sub-agent)
    │   └─ Reads ai-docs-map.json → returns doc file paths
    │
    ├─► Service Locator (sub-agent)
    │   └─ Reads ai-service-map.json → returns source file paths
    │
    └─► Research Agent reads files, follows references,
        re-queries sub-agents if needed (max 3 iterations),
        then synthesizes the final answer
```
