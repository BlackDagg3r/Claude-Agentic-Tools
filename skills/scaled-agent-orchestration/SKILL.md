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
RUN_ID="$(date +%Y%m%d-%H%M%S)-$(openssl rand -hex 4)"
mkdir -p "/tmp/orchestration/$RUN_ID"/{workers,synthesis} || { echo "ERROR: failed to create run directory"; exit 1; }
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
   # Replace the file list below with your actual worker IDs (e.g., a1.md a2.md b1.md b2.md)
   for f in /tmp/orchestration/$RUN_ID/workers/a1.md /tmp/orchestration/$RUN_ID/workers/a2.md; do
     [ -f "$f" ] || { echo "MISSING: $f"; continue; }
     tail -1 "$f" | grep -q "DONE" || echo "INCOMPLETE: $f"
   done
   ```
3. **Timeout:** If a worker has not returned after 5 minutes, treat it as failed (missing DONE sentinel). Do not wait indefinitely.
4. If any worker failed or timed out: retry once with the same prompt, or note it as a gap for the coordinator
5. Only proceed to coordinators when all workers are done or accounted for

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

### Template Variables

Both prompt templates use `{placeholder}` syntax. Replace before dispatching:

| Placeholder | Used in | Example value |
|-------------|---------|---------------|
| `{specific_task_description}` | worker | "Scan SKILL.md for broken cross-references" |
| `{output_file_path}` | worker | `/tmp/orchestration/$RUN_ID/workers/a1.md` |
| `{allowed_tools}` | worker | `Read, Grep, Glob` |
| `{task_name}` | worker | "Cross-Reference Check" |
| `{scope_constraint}` | worker | "Only files in skills/scaled-agent-orchestration/" |
| `{explicit_file_list}` | coordinator | A markdown list: `- /tmp/.../workers/a1.md\n- /tmp/.../workers/a2.md` |
| `{synthesis_file_path}` | coordinator | `/tmp/orchestration/$RUN_ID/synthesis/security.md` |
| `{domain_name}` | coordinator | "Security" |
| `{worker_N_name}` | coordinator | "Worker A1" (matches worker IDs in your plan) |

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
| 1 coordinator summary | ~400 tokens (under 300 words) |
| 30 workers + 4 coordinators | ~2,200 tokens total |
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
- File paths that escape `/tmp/orchestration/{RUN_ID}/` — always verify paths stay within the run directory

## Common Mistakes

**Workers return full results:** Tell workers to write to file. Return only "Done. {summary}."

**Too many coordinators:** 3-5 is the sweet spot. More defeats the purpose.

**No wave batching:** 30 simultaneous background agents can still strain the system. Batch 8 at a time across all domains.

**Coordinators read wrong files:** Use explicit file paths in coordinator prompts. Don't rely on glob patterns.

**No completion verification:** Always verify worker output files exist and contain `<!-- DONE -->` before dispatching coordinators.

**Silent worker failures:** Workers that crash produce no output. Check for missing files before coordinator dispatch. Retry once, then flag as gap.

## Build Pipeline (v2.0)

For building enterprise applications, not just auditing. Uses the same file-based communication pattern but adds isolation (git worktrees), contracts, QC gates, persistent memory, and training data generation.

### When to Use Build Mode

| Request type | Mode |
|-------------|------|
| "Audit/review/scan this codebase" | v1.1 audit (above) |
| "Build a new application" | v2.0 build pipeline |
| "Add a major feature to existing app" | v2.0 build pipeline |

### Build Pipeline Stages

```
Stage 1:  ARCHITECT   (opus, foreground, human approves)
Stage 1b: BOOTSTRAP   (sonnet, generate training data from blueprint)
Stage 2:  SCAFFOLD    (opus, foreground, master QC)
Stage 3:  BUILD       (sonnet/opus, parallel in worktrees, per-module QC)
Stage 4:  ASSEMBLE    (sonnet, sequential merge, integration tests)
Stage 5:  HARDEN      (opus, v1.1 fan-out audit pattern)
Stage 6:  SHIP        (sonnet, human approves)
Stage 7:  CURATE      (sonnet, memory maintenance)
```

### Directory Setup (Build Mode)

```bash
RUN_ID="$(date +%Y%m%d-%H%M%S)-$(openssl rand -hex 4)"
mkdir -p "/tmp/orchestration/$RUN_ID"/{workers,synthesis,builds,qc,conflicts} || { echo "ERROR: failed to create run directory"; exit 1; }
mkdir -p "{project_root}/.orchestration"/{contracts/interfaces,contracts/decisions,brains,training} || { echo "ERROR: failed to create orchestration directory"; exit 1; }
```

### Stage 1: Dispatch Architect

```
Task tool call:
  prompt: "[filled architect-prompt.md]"
  subagent_type: "general-purpose"
  model: "opus"
  run_in_background: false
```

**Gate:** Present architecture to human. Wait for approval before proceeding.

### Stage 1b: Bootstrap Training Data

```
Task tool call:
  prompt: "[filled training-generator-prompt.md with all contracts/ files]"
  subagent_type: "general-purpose"
  model: "sonnet"
  run_in_background: false
```

### Stage 2: Dispatch Scaffold Agent

Use a general-purpose opus agent to create the project skeleton. The scaffold MUST match modules.json exactly.

**Gate:** Master agent (or parent) verifies scaffold matches contracts.

### Stage 3: Dispatch Builders (Parallel)

For each module in modules.json:

```
Task tool call:
  prompt: "[filled builder-prompt.md for this module]"
  subagent_type: "general-purpose"
  model: "{builder_model from modules.json}"
  run_in_background: true
  isolation: "worktree"
```

Wave limits apply (same as v1.1: opus 4, sonnet 8, haiku 12).

After each builder returns, dispatch a QC agent:

```
Task tool call:
  prompt: "[filled qc-prompt.md for this module]"
  subagent_type: "general-purpose"
  model: "sonnet"
  run_in_background: false
```

**Gate:** QC verdict determines next step:
- PASS → module queued for assembly
- FAIL-CRITICAL → halt pipeline
- FAIL-MAJOR → builder gets fix cycle (re-dispatch with QC findings)
- MINOR findings → logged, no block

### Stage 4: Dispatch Assembler

```
Task tool call:
  prompt: "[filled assembler-prompt.md with ordered module list]"
  subagent_type: "general-purpose"
  model: "sonnet"
  run_in_background: false
```

### Stage 5: Harden (uses v1.1 audit pattern)

Dispatch v1.1 orchestration against the integrated codebase:
- Domain A: Security audit workers + coordinator
- Domain B: Performance analysis workers + coordinator
- Domain C: Edge case / compliance workers + coordinator

Same rules as v1.1: wave batched, file-based, coordinator synthesis.

### Stage 6: Ship

Present hardening results to human. If approved, merge integration branch to main.

### Stage 7: Curate

After each work session, update brain files, compress memory, regenerate training data.

### Contract Conflict Protocol

If a builder returns "BLOCKED. Contract conflict:":
1. Read the conflict file at `.orchestration/conflicts/{module}-{issue}.md`
2. Evaluate: can the contract flex, or must the builder adapt?
3. Present to human for approval
4. If contract changes: update contracts/ and notify affected builders via brain files
5. Resolved conflict becomes a new ADR in contracts/decisions/

### Trust Flag System

Decisions in locks.json carry lock levels:
- `unlocked` — any agent can discuss
- `locked` — only locking domain + master + human can change
- `critical` — only human + master can change

Builders check locks.json before implementation. If a decision is locked or critical, they comply without question. The assertion field tells them what to implement.

### Memory Layer Reference

| Layer | Location | Updated by | Contains |
|-------|----------|-----------|----------|
| L1 Brain Files | `.orchestration/brains/{role}.md` | Producing agent | Current state, decisions, next steps |
| L2 Session Summaries | Pinecone `project-{name}/{role}` | Producing agent | Recent session history |
| L3 Deep Archive | Pinecone `project-{name}/archive` | Stage 7 curator | Historical rationale |
| L4 Training Data | `.orchestration/training/*.jsonl` | Every agent | Q/A pairs for fine-tuning |

### Build Pipeline Template Variables

| Placeholder | Used in | Example |
|-------------|---------|---------|
| `{existing_codebase_summary_or_none}` | architect | "None" or a brief summary of existing code |
| `{project_description}` | architect | "SaaS billing platform with Stripe integration" |
| `{project_root}` | all build templates | `/Users/atwellguy/my-project` |
| `{project_name}` | training generator | "billing-platform" |
| `{module_name}` | builder, qc | "auth" |
| `{module_description}` | builder | "Authentication and authorization" |
| `{module_code_path}` | qc | `/tmp/orchestration/{RUN_ID}/builds/module-auth/` |
| `{qc_report_path}` | builder, qc | `/tmp/orchestration/{RUN_ID}/qc/auth-build-qc.md` |
| `{integration_branch_name}` | assembler | `orchestration/{RUN_ID}/integration` |
| `{ordered_module_list_with_branches}` | assembler | Numbered list of modules with branch names |
| `{test_command}` | qc | `pytest tests/ -v` or `npm test` |
| `{integration_test_command}` | assembler | `pytest tests/integration/ -v` |
| `{qc_dir}` | assembler | `/tmp/orchestration/{RUN_ID}/qc` |
| `{assembly_report_path}` | assembler | `/tmp/orchestration/{RUN_ID}/synthesis/assembly.md` |
| `{training_output_path}` | training gen, builder | `.orchestration/training/bootstrap.jsonl` |
| `{source_file_list}` | training gen | Markdown list of all contracts/ files |
| `{start_pair_id}` | training gen | `1` (bootstrap) or next available ID |
| `{allowed_tools}` | builder | `Read, Write, Edit, Bash, Glob, Grep` |
