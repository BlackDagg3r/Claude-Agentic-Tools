---
name: scaled-agent-orchestration
description: Use when spawning 10+ agents, when previous agent runs caused context overflow or compaction errors, when hierarchical or tiered agent orchestration is needed, or when parallel fan-out requires more agents than can safely return results to one context window
---

# Scaled Agent Orchestration

Run 30+ agents without blowing context. Workers write to disk, coordinators synthesize, parent receives only brief summaries.

**Core principle:** Never let more than 5 agent results flow into one context window. Use files as the communication bus.

## When to Use

| Agents needed | Approach |
|---------------|----------|
| < 5 | Just dispatch them directly |
| 5-10 | Use dispatching-parallel-agents if available, otherwise dispatch directly |
| 10+ | **Use this skill** |

**Use when:**
- Need 10+ parallel agents
- Previously hit "Conversation too long" or compaction errors
- Work decomposes into independent domains with sub-tasks

**Don't use when:**
- < 10 agents needed
- Tasks are tightly coupled (agents need each other's output)
- Sequential dependency chain

## The Architecture

```
Parent context (receives ~5 short summaries)
├── Coordinator A (reads worker files, returns 1 paragraph)
│   ├── Worker A1 → /tmp/orchestration/{run}/workers/a1.md
│   ├── Worker A2 → /tmp/orchestration/{run}/workers/a2.md
│   └── Worker A3 → /tmp/orchestration/{run}/workers/a3.md
├── Coordinator B
│   ├── Worker B1 → /tmp/orchestration/{run}/workers/b1.md
│   └── Worker B2 → /tmp/orchestration/{run}/workers/b2.md
└── Coordinator C
    └── ...
```

## The Process

### 1. Setup output directory

```bash
RUN_ID=$(date +%Y%m%d-%H%M%S)-$RANDOM
mkdir -p /tmp/orchestration/$RUN_ID/{workers,synthesis}
```

### 2. Decompose into domains

Group work into 3-5 coordinator domains. Each domain gets 3-8 workers.

### 3. Dispatch workers (background, wave by wave)

Use the **Task tool** with `run_in_background: true` to dispatch each worker. Each worker writes findings to its output file and returns only a one-sentence summary.

**Dispatch example:**
```
Task tool call:
  prompt: "[filled worker-prompt.md template]"
  subagent_type: "general-purpose"  (or a specific agent type)
  model: "haiku"                    (or "sonnet"/"opus" per model selection below)
  run_in_background: true
```

Each worker prompt MUST include:
- Specific narrow task
- Output file path (e.g., `/tmp/orchestration/{RUN_ID}/workers/a1.md`)
- Instruction to write `<!-- DONE -->` as the last line of output
- Instruction to return ONLY "Done. {one sentence}" to parent

**Wave limit: 8 workers maximum across ALL domains simultaneously** — not 8 per domain. For different model tiers, adjust:
- haiku workers: up to 12 per wave
- sonnet workers: up to 8 per wave
- opus workers: up to 4 per wave

See `./worker-prompt.md` for the full template.

### 3b. Verify worker completion

**Before dispatching coordinators, verify ALL workers have finished:**

1. Wait for all background Task agents to return their "Done." or "Failed." messages
2. Check that each expected output file exists and contains `<!-- DONE -->` as its last line:
   ```bash
   for f in /tmp/orchestration/$RUN_ID/workers/{expected_files}; do
     tail -1 "$f" | grep -q "DONE" || echo "INCOMPLETE: $f"
   done
   ```
3. If any worker failed: retry once with the same prompt, or note it as a gap for the coordinator
4. Only proceed to coordinators when all workers are done or accounted for

### 4. Dispatch coordinators (after ALL workers verified)

Each coordinator reads its workers' output files (provided as an **explicit list of file paths**, not glob patterns) and writes a synthesis. Returns only a brief summary to parent.

**Dispatch example:**
```
Task tool call:
  prompt: "[filled coordinator-prompt.md template with explicit file list]"
  subagent_type: "general-purpose"
  model: "sonnet"
```

Coordinators run in **foreground** (not background) so the parent receives their summaries directly.

See `./coordinator-prompt.md` for the full template.

### 5. Parent decides next action

Parent has 3-5 coordinator summaries (small context cost). Full details on disk if needed.

## Wave Strategy

For 30+ agents, batch workers in waves (8 max across all domains simultaneously):

```
Wave 1: Workers A1-A4, B1-B4 (background) → verify completion
Wave 2: Workers A5-A8, C1-C4 (background) → verify completion
Wave 3: Coordinators A, B, C (foreground) → summaries to parent
Wave 4: Action agents based on findings
```

## Token Budget

| Component | Context cost to parent |
|-----------|----------------------|
| 1 worker (background, file output) | ~20 tokens ("Done. Found 3 issues.") |
| 1 coordinator summary | ~200 tokens |
| 30 workers + 4 coordinators | ~1,400 tokens total |
| 30 agents flat (anti-pattern) | ~150,000 tokens (context overflow) |

## Model Selection

Choose the model for each agent based on task type. Do NOT default everything to the same model. Pass `model: "haiku"`, `model: "sonnet"`, or `model: "opus"` in the Task tool call.

| Task type | Model |
|-----------|-------|
| Simple: search, grep, lookup, exists-check, formatting | haiku |
| Moderate: code review, test writing, pattern analysis, synthesis | sonnet |
| Complex: architecture, security audit, novel design, writing code | opus |

### By agent role

| Role | Model | Why |
|------|-------|-----|
| **Workers** (search, scan, grep) | haiku | Narrow scope, fast, cheap |
| **Workers** (code analysis, review) | sonnet | Needs reasoning but scope is bounded |
| **Workers** (security audit, architecture) | opus | Accuracy critical, complex judgment |
| **Coordinators** (synthesize files) | sonnet | Reading + ranking, moderate reasoning |
| **Action agents** (implement fixes) | opus | Writing code, needs full capability |

**Note:** If a "simple" task requires a multi-step tool chain (glob → grep → read → filter), use **sonnet** not haiku. Haiku's tool-use reliability drops with complex chains.

### Quick rule

- **Reading** → haiku
- **Analyzing** → sonnet
- **Deciding or writing code** → opus

## Red Flags - STOP

- Spawning 10+ agents without file-based output
- Workers returning verbose results to parent
- Skipping coordinator layer (flat fan-out at scale)
- All agents in foreground (use background for workers)
- No output directory setup before dispatching
- Dispatching coordinators before verifying all workers finished
- Using glob patterns instead of explicit file paths in coordinator prompts

## Common Mistakes

**Workers return full results:** Tell workers to write to file. Return only "Done. {summary}."

**Too many coordinators:** 3-5 is the sweet spot. More defeats the purpose.

**No wave batching:** 30 simultaneous background agents can still strain the system. Batch 8 at a time across all domains.

**Coordinators read wrong files:** Use explicit file paths in coordinator prompts. Don't rely on glob patterns.

**No completion verification:** Always verify worker output files exist and contain `<!-- DONE -->` before dispatching coordinators.

**Silent worker failures:** Workers that crash produce no output. Check for missing files before coordinator dispatch. Retry once, then flag as gap.
