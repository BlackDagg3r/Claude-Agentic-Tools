# Scaled Agent Orchestration

Claude Code plugin for running 10-30+ parallel agents without blowing the 200k context window.

## The Problem

Spawning 30 agents flat dumps ~150,000 tokens back into the parent context, causing "Conversation too long" compaction errors and session death.

## The Solution

Tiered file-based communication:
- **Workers** write findings to disk, return only "Done. {one sentence}" (~20 tokens each)
- **Coordinators** read worker files, return top 3 findings under 300 words (~400 tokens each)
- **Parent** receives 3-5 brief summaries instead of 30 full reports

| Setup | Context cost |
|-------|-------------|
| 30 workers + 4 coordinators (this plugin) | ~2,200 tokens |
| 30 agents flat (anti-pattern) | ~150,000 tokens |

## Installation

**Via Git marketplace (recommended):**

```bash
claude plugin marketplace add https://github.com/BlackDagg3r/Claude-Agentic-Tools
claude plugin install scaled-agent-orchestration@blackdagger-tools
```

This clones the marketplace repo to `~/.claude/plugins/marketplaces/`. The `marketplace add` command handles the git clone automatically.

**Via local directory (session only):**

```bash
claude --plugin-dir ~/scaled-agent-orchestration
```

## Usage

```
/scaled-agent-orchestration <task description>
/scaled-agent-orchestration setup
```

Or describe a task needing 10+ agents — the skill triggers automatically.

## Model Selection

The plugin chooses the right model per agent:

| Task type | Model |
|-----------|-------|
| Search, grep, lookup | haiku (fast, cheap) |
| Code review, synthesis | sonnet (balanced) |
| Architecture, security audit, writing code | opus (full capability) |

**Coordinators always use sonnet.** Workers use haiku/sonnet/opus based on task complexity.

## Wave Limits

Workers batch in waves with per-model limits:

| Model | Max per wave |
|-------|-------------|
| haiku | 12 |
| sonnet | 8 |
| opus | 4 |

The limit applies across ALL domains simultaneously — not per domain. Each wave completes and is verified (DONE sentinel check) before the next starts.

## Architecture

```
Parent (receives ~5 summaries)
├── Coordinator A (reads worker files, returns 1 paragraph)
│   ├── Worker A1 → /tmp/orchestration/{run}/workers/a1.md
│   ├── Worker A2 → /tmp/orchestration/{run}/workers/a2.md
│   └── Worker A3 → /tmp/orchestration/{run}/workers/a3.md
├── Coordinator B
│   └── ...
└── Coordinator C
    └── ...
```

Output directories: `/tmp/orchestration/{RUN_ID}/workers/` and `/tmp/orchestration/{RUN_ID}/synthesis/`

Prompt templates: see `skills/scaled-agent-orchestration/worker-prompt.md` and `coordinator-prompt.md` for the full worker and coordinator prompt templates with all placeholder variables.

## License

MIT
