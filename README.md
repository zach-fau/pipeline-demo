# AI Research Agent Pipeline

A system of Claude Code slash commands that lets you ask questions about your codebase in plain English, then take a task from research all the way to reviewed, implemented code — without manual file searching or context overload.

## The Pipeline

```
/generate-docs  →  /generate-maps  →  /research  →  /propose  →  /plan  →  /implement  →  /review
      ↓                  ↓                ↓             ↓           ↓           ↓              ↓
  YAML docs        JSON indexes      Research.md   Proposal.md  Plan.md    Code changes    Review.md
                                                                           + Log.md
```

Each agent starts with a clean context. The only information that crosses boundaries is what's written to disk. Sub-agents are re-spawned at each step to re-discover relevant files.

## Installation

Copy the command files to your Claude Code commands directory:

```bash
cp 1-commands/*.md ~/.claude/commands/
```

That's it — the commands are now available in every project via `/command-name`.

## Usage

### Phase 1: Initial Setup (one-time per project)

```bash
/generate-docs          # Analyzes codebase, creates structured YAML docs per service
/generate-maps          # Builds JSON index files for fast lookup
```

### Phase 2: Research (whenever you need to understand something)

```bash
/research "How does authentication work?"
/research "What happens when a user places an order?"
```

The research agent spawns sub-agents to find relevant docs and source files, reads them, follows cross-references iteratively (up to 3 rounds), and writes a comprehensive report to `documentation/Research.md`.

### Phase 3: Propose → Plan → Implement → Review

Once you have a research report, you can take it all the way to code:

```bash
/propose                    # Evaluates 1-3 approaches, recommends one → Proposal.md
/plan                       # Reads every affected file, produces ordered change specs → Plan.md
/implement                  # Executes changes step-by-step → code changes + Implementation-Log.md
/review                     # Verifies implementation against the plan → Review.md
```

Each step reads the previous step's output. You can stop at any point — `/research` alone is useful for questions, `/propose` is useful for evaluating approaches without committing to code.

#### Options

| Command | Arguments | Example |
|---------|-----------|---------|
| `/propose` | _(none)_ | `/propose` |
| `/plan` | Approach number | `/plan approach 2` |
| `/implement` | Step number to resume from | `/implement step 3` |
| `/review` | Focus areas | `/review` |

#### Handling Long Implementations

The implementer monitors its context usage and stops cleanly when needed:
1. Finishes the current step
2. Writes progress to `Implementation-Log.md` with a "Next Steps" section
3. Tells you how to resume

Run `/implement` again — it picks up where it left off automatically by reading the existing log.

#### Error Handling

The implementer stops and reports when it hits issues (file doesn't match plan, missing dependencies, etc.) rather than attempting autonomous recovery. This keeps the implementation log accurate and lets you decide how to proceed.

## What Each Folder Contains

### 1-commands/

The agent command files that make the system work:

| File | Purpose |
|------|---------|
| `generate-docs.md` | Analyzes your codebase and generates YAML docs (openapi, internalfunctions, events) per service |
| `generate-maps.md` | Builds the two JSON index files from docs and source code |
| `documentation-locator.md` | Sub-agent that searches ai-docs-map.json to find relevant doc paths |
| `service-locator.md` | Sub-agent that searches ai-service-map.json to find relevant source file paths |
| `research.md` | Research agent — orchestrates sub-agents, follows references, synthesizes answers |
| `propose.md` | Proposal agent — evaluates approaches with tradeoffs, recommends one |
| `plan.md` | Planning agent — reads affected files, produces exact change specifications |
| `implement.md` | Implementer agent — executes plan step-by-step, tracks progress in a log |
| `review.md` | Review agent — validates implementation against plan, checks consumers |
| `setup.md` | Quick-start guide for the full system |

### 2-generated-docs/

Example output from the documentation generator (from the Z Cafe test project):

- `openapi/` — API endpoints, routes, parameters for each service
- `internalfunctions/` — Helper functions, business logic, utilities
- `events/` — Events emitted and listened to

### 3-generated-maps/

Example mapping files:

- `ai-docs-map.json` — Maps service names and keywords to documentation file paths
- `ai-service-map.json` — Maps routes, actions, and source files for fast lookup
- `ai-config.json` — Configuration for the AI research system

### 4-research-output/

Example research results showing what the system produces when you ask it questions.

## Architecture

```
/generate-docs  →  documentation/*.yml  (structured docs per service)
/generate-maps  →  ai-docs-map.json + ai-service-map.json  (indexes)

/research "question"
    ├─► Documentation Locator (sub-agent)
    │   └─ Reads ai-docs-map.json → returns doc file paths
    ├─► Service Locator (sub-agent)
    │   └─ Reads ai-service-map.json → returns source file paths
    └─► Reads files, follows references (max 3 iterations) → Research.md

/propose
    ├─► Reads Research.md
    ├─► Spawns sub-agents (re-discovers relevant files)
    ├─► Reads source files, evaluates approaches → Proposal.md

/plan
    ├─► Reads Proposal.md + Research.md
    ├─► Spawns sub-agents (service-specific queries)
    ├─► Reads ALL files to be modified (exact current state) → Plan.md

/implement
    ├─► Reads Plan.md (+ Implementation-Log.md if resuming)
    ├─► For each step: spawns sub-agents, reads file, makes edit
    ├─► Stops cleanly on errors or context limits → Implementation-Log.md

/review
    ├─► Reads Plan.md + Implementation-Log.md
    ├─► Spawns sub-agents (finds modified files + consumers)
    ├─► Reads each file, compares against plan → Review.md
```

### Design Principles

- **No context overload** — Sub-agents return file paths, not file contents. Each pipeline stage starts fresh.
- **Disk-based communication** — Agents only share information through Markdown files in `documentation/`. No in-memory state crosses boundaries.
- **Iterative discovery** — The research agent follows dependency chains. The proposal and plan agents re-discover relevant files for their specific needs.
- **Graceful degradation** — The implementer stops cleanly and logs progress. The reviewer handles incomplete implementations.
- **Reusable** — Works across any codebase once configured. Language and framework agnostic.

## Artifacts Produced

After a full pipeline run, your `documentation/` folder will contain:

| File | Produced By | Contents |
|------|-------------|----------|
| `Research.md` | `/research` | Comprehensive answer to the research question |
| `Proposal.md` | `/propose` | 1-3 approaches with tradeoffs and a recommendation |
| `Plan.md` | `/plan` | Ordered change specifications with exact file-level detail |
| `Implementation-Log.md` | `/implement` | Record of every change made, with status per step |
| `Review.md` | `/review` | Per-file verification with PASS / PASS WITH NOTES / NEEDS FIXES verdict |

---

Built with Claude Code. Tested on the Z Cafe React Native app.
