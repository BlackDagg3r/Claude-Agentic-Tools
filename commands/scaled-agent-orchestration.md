---
name: scaled-agent-orchestration
description: Set up and run a tiered agent orchestration with file-based communication for 10+ parallel agents
allowed-tools: Bash, Read, Write, Glob, Grep, Task
argument-hint: "<task description or 'setup'>"
---

Run a scaled agent orchestration using tiered file-based communication.

**If $ARGUMENTS is empty or equals "setup":**
1. Create the output directory structure:
   ```bash
   RUN_ID=$(date +%Y%m%d-%H%M%S)-$RANDOM
   mkdir -p /tmp/orchestration/$RUN_ID/{workers,synthesis}
   ```
2. Tell the user the run directory and ask them what work to decompose.

**If $ARGUMENTS contains a task description:**
1. Create the output directory (use `RUN_ID=$(date +%Y%m%d-%H%M%S)-$RANDOM`)
2. Decompose the task into 3-5 coordinator domains
3. For each domain, identify 3-8 worker tasks
4. Present the plan as a table showing: domain, worker ID, task, model (haiku/sonnet/opus), output file path
5. Get user confirmation before dispatching
6. Execute the orchestration following ALL rules below

**Orchestration rules — follow these precisely:**

1. **Workers dispatch via Task tool** with `run_in_background: true`. Choose model per task: `model: "haiku"` for search/grep, `model: "sonnet"` for analysis, `model: "opus"` for architecture/security.
2. **Workers write to files** at `/tmp/orchestration/{RUN_ID}/workers/{id}.md` and return ONLY "Done. {one sentence}" to parent.
3. **Workers write `<!-- DONE -->` as the last line** of their output file (sentinel marker).
4. **Wave limit: 8 workers max across all domains simultaneously.** Haiku can go to 12, opus limit to 4.
5. **Verify completion before coordinators:** After each wave, confirm all expected output files exist and contain `<!-- DONE -->`. Retry failed workers once.
6. **Coordinators dispatch via Task tool** in foreground (not background) with `model: "sonnet"`. Give each coordinator an **explicit list of file paths** (not glob patterns).
7. **Coordinators write synthesis** to `/tmp/orchestration/{RUN_ID}/synthesis/{domain}.md` and return only top 3 findings (under 300 words) to parent.
8. **Never let more than 5 agent results flow into one context window.**
9. After all coordinators report, summarize findings and propose next actions.
