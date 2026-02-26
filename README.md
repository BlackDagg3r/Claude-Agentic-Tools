# Scaled Agent Orchestration

Claude Code plugin for running 10-30+ parallel agents without blowing the 200k context window.

## The Problem

Spawning 30 agents flat dumps ~150,000 tokens back into the parent context, causing "Conversation too long" compaction errors and session death.

## The Solution

Tiered file-based communication:
- **Workers** write findings to disk, return only "Done. {one sentence}" (~20 tokens each)
- **Coordinators** read worker files, return top 3 findings (~200 tokens each)
- **Parent** receives 3-5 brief summaries instead of 30 full reports

| Setup | Context cost |
|-------|-------------|
| 30 workers + 4 coordinators (this plugin) | ~1,400 tokens |
| 30 agents flat (anti-pattern) | ~150,000 tokens |

## Installation

**Via Git marketplace (recommended):**

```bash
claude plugin marketplace add https://github.com/BlackDagg3r/Claude-Agentic-Tools
claude plugin install scaled-agent-orchestration@blackdagger-tools
```

**Note:** This marketplace must be added via git clone, not via direct URL to marketplace.json.

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

Workers batch in waves of 8. Each wave completes and is verified before the next starts.

## License

MIT
