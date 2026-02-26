---
name: scaled-orchestration
description: Set up and run a tiered agent orchestration with file-based communication for 10+ parallel agents
allowed-tools: Bash, Read, Write, Glob, Grep, Task
argument-hint: "<task description or 'setup'>"
---

You have been asked to run a scaled agent orchestration. Load the scaled-agent-orchestration skill and follow it precisely.

**If the user said "setup" or gave no arguments:**
1. Create the output directory structure:
   ```bash
   RUN_ID=$(date +%Y%m%d-%H%M%S)
   mkdir -p /tmp/orchestration/$RUN_ID/{workers,synthesis}
   ```
2. Tell the user the run directory and ask them what work to decompose.

**If the user described a task:**
1. Create the output directory
2. Decompose the task into 3-5 coordinator domains
3. For each domain, identify 3-8 worker tasks
4. Present the plan as a table and get confirmation
5. Execute following the skill: workers in background writing to files, then coordinators synthesizing

**Key rules from the skill:**
- Workers use `run_in_background: true` and write to `/tmp/orchestration/{RUN_ID}/workers/`
- Workers return ONLY "Done. {one sentence}" to parent
- Batch workers in waves of 8 max
- Coordinators read worker files and write synthesis to `/tmp/orchestration/{RUN_ID}/synthesis/`
- Coordinators return only top 3 findings (under 300 words) to parent
- Never let more than 5 agent results flow into one context window
