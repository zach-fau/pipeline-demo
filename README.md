# AI Research Agent Pipeline

A system for asking questions about your codebase in plain English and getting comprehensive answers without manual file searching or context overload.

## What We Built

We filled all the gaps from the original requirements. Here's what each folder contains:

### 📁 1-commands/
The agent configurations that make the system work:

- **`generate-docs.md`** - Prompts for Claude Code to analyze your codebase and generate the three YAML files (openapi, internalfunctions, events) for each service
- **`generate-maps.md`** - Prompts to scan your codebase/docs and create the two mapping files (ai-docs-map.json, ai-service-map.json)
- **`documentation-locator.md`** - Sub-agent that searches ai-docs-map.json to find relevant documentation paths
- **`service-locator.md`** - Sub-agent that searches ai-service-map.json to find relevant source file paths
- **`research.md`** - Main research agent that orchestrates the sub-agents and synthesizes answers
- **`setup.md`** - Quick-start guide to configure everything in your Claude Code environment

### 📁 2-generated-docs/
Example output from the documentation generator (from our Z Cafe test project):

- `openapi/` - API endpoints, routes, parameters for each service
- `internalfunctions/` - Helper functions, business logic, utilities
- `events/` - Events emitted and listened to

### 📁 3-generated-maps/
Example mapping files:

- **`ai-docs-map.json`** - Maps service names → their three documentation files
- **`ai-service-map.json`** - Maps routes/actions → actual source code file paths
- **`ai-config.json`** - Configuration for the AI research system

### 📁 4-research-output/
Example research results showing what the system produces when you ask it questions.

## How It Works

**Phase 1: Initial Setup** (one-time per project)
```bash
/generate-docs          # Analyzes codebase, creates YAML docs
/generate-maps          # Creates the mapping files
```

**Phase 2: Daily Use** (whenever you need to understand something)
```bash
/research "How does review pagination work?"
```

The research agent:
1. Calls DocumentationLocator to find relevant docs
2. Calls ServiceLocator to find relevant source files
3. Reads those specific files
4. Discovers dependencies and loops until complete
5. Synthesizes everything into a comprehensive answer

## The Architecture

- **No context overload** - Sub-agents return file paths, not content
- **Iterative discovery** - Research agent follows dependency chains
- **Comprehensive answers** - All relevant code and docs are found automatically
- **Reusable** - Works across any codebase once configured

---

Built with Claude Code. Tested on the Z Cafe React Native app.
