---
description: Quick-start guide for the AI Research Agent system
---

# AI Research Agent — Setup Guide

This system lets you ask plain-English questions about any codebase, get comprehensive answers, and then take a task all the way from research to implemented code. It works by generating structured documentation, building index maps, and using specialized agents that communicate through disk artifacts.

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
```

The research agent writes its findings to `documentation/Research.md`.

## The Full Pipeline

Once you have a research report, you can take it all the way to implemented code:

```
/research "question"  →  /propose  →  /plan  →  /implement  →  /review
       ↓                    ↓           ↓           ↓              ↓
  Research.md          Proposal.md   Plan.md    Code changes    Review.md
                                                + Log.md
```

Each step clears context. Agents communicate only through disk artifacts.

### Step-by-step:

1. **`/research "your question"`** — Investigate the codebase, write `Research.md`
2. **`/propose`** — Read Research.md, evaluate 1-3 approaches, recommend one, write `Proposal.md`
3. **`/plan`** — Read Proposal.md, read all affected source files, produce detailed change specs in `Plan.md`
4. **`/implement`** — Read Plan.md, execute changes step-by-step, write `Implementation-Log.md`
5. **`/review`** — Read Plan.md + Log, verify each file against the plan, write `Review.md`

### Options

- **`/propose`** — uses Research.md as-is
- **`/plan`** — uses recommended approach; pass `approach 2` to plan a different one
- **`/implement`** — starts from Step 1; pass `step 3` to resume from a specific step
- **`/review`** — reviews all completed work; pass additional focus areas as needed

### Handling long implementations

The implementer agent monitors its context usage and stops cleanly when needed:
1. Finishes the current step
2. Writes progress to `Implementation-Log.md` with a "Next Steps" section
3. Tells you how to resume

Just run `/implement` again — it picks up where it left off automatically.

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
    ├── Research.md              # Latest research report
    ├── Proposal.md             # Approach evaluation
    ├── Plan.md                 # Detailed change specifications
    ├── Implementation-Log.md   # Record of changes made
    └── Review.md               # Validation against plan
```

## Tips

- **Be specific with questions.** "How does auth work?" is good. "How does the login flow validate credentials and create a session?" is better.
- **Regenerate docs after major changes.** The agents' quality depends on up-to-date documentation and maps.
- **Use service-specific regeneration.** `/generate-docs Auth` is faster than regenerating everything when you only changed auth code.
- **Check the artifacts.** Each step writes a Markdown file you can review before proceeding to the next step.
- **You don't have to run the full pipeline.** Use `/research` on its own for questions, or stop after `/propose` if you just want to evaluate approaches.

## Architecture

```
/generate-docs  →  documentation/*.yml  (structured docs per service)
/generate-maps  →  ai-docs-map.json + ai-service-map.json  (indexes)

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
        then synthesizes the final answer → Research.md

/propose
    │
    ├─► Reads Research.md (link to previous step)
    ├─► Spawns sub-agents (re-discovers relevant files)
    ├─► Reads source files, evaluates approaches
    └─► Writes Proposal.md

/plan
    │
    ├─► Reads Proposal.md + Research.md
    ├─► Spawns sub-agents (service-specific queries)
    ├─► Reads ALL files to be modified (exact current state)
    └─► Writes Plan.md (ordered change specifications)

/implement
    │
    ├─► Reads Plan.md (+ Implementation-Log.md if resuming)
    ├─► For each step: spawns sub-agents, reads file, makes edit
    ├─► Stops cleanly on errors or context limits
    └─► Writes Implementation-Log.md

/review
    │
    ├─► Reads Plan.md + Implementation-Log.md
    ├─► Spawns sub-agents (finds modified files + consumers)
    ├─► Reads each modified file, compares against plan
    └─► Writes Review.md (PASS / PASS WITH NOTES / NEEDS FIXES)
```

Each agent starts with a clean context. The ONLY information that crosses boundaries is what's written to the `documentation/` directory. Sub-agents are re-spawned at each step to re-discover relevant files for that step's specific needs.
