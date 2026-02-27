---
name: build-pipeline
description: Build an enterprise application using the 7-stage orchestrated pipeline with parallel agents, QC gates, and persistent memory
allowed-tools: Bash, Read, Write, Edit, Glob, Grep, Task
argument-hint: "<project description>"
---

Run the enterprise build pipeline using tiered agent orchestration.

**If $ARGUMENTS is empty:**
1. Ask the user what they want to build.

**If $ARGUMENTS contains a project description:**

Execute the 7-stage build pipeline. Follow ALL stages in order. Do NOT skip stages.

**Stage 1: ARCHITECT**
1. Create orchestration directories:
   ```bash
   RUN_ID="$(date +%Y%m%d-%H%M%S)-$(openssl rand -hex 4)"
   PROJECT_ROOT="$(pwd)"
   mkdir -p "/tmp/orchestration/$RUN_ID"/{workers,synthesis,builds,qc,conflicts} || { echo "ERROR"; exit 1; }
   mkdir -p "$PROJECT_ROOT/.orchestration"/{contracts/interfaces,contracts/decisions,brains,training} || { echo "ERROR"; exit 1; }
   ```
2. Dispatch architect agent (opus, foreground) with filled `architect-prompt.md`
3. Present the architecture to user for approval
4. **STOP and WAIT for human approval before proceeding**

**Stage 1b: BOOTSTRAP TRAINING DATA**
1. Dispatch training data generator (sonnet, foreground) with all contracts/ files
2. Verify: output file exists, pair count > 50

**Stage 2: SCAFFOLD**
1. Dispatch scaffold agent (opus, foreground) to create project structure
2. Verify scaffold matches modules.json (all module directories exist, shared types created)

**Stage 3: BUILD (parallel)**
1. Read modules.json for module list and build order
2. For each module, dispatch builder agent with `isolation: "worktree"` and `run_in_background: true`
3. Wave limit: opus 4, sonnet 8 max across all modules simultaneously
4. After each builder returns, dispatch QC agent (sonnet, foreground)
5. Handle QC verdicts:
   - PASS → queue for assembly
   - FAIL-CRITICAL → halt pipeline, report to user
   - FAIL-MAJOR → re-dispatch builder with QC findings (max 2 retries)

**Stage 4: ASSEMBLE**
1. Dispatch assembler agent (sonnet, foreground) with ordered list of passed modules
2. If merge conflicts or test failures: report to user

**Stage 5: HARDEN**
1. Run v1.1 audit orchestration against integrated codebase
2. Security, performance, edge cases, compliance domains
3. Present coordinator summaries to user
4. Handle findings by severity (critical stops, major gets fix cycle)

**Stage 6: SHIP**
1. Generate docs, CHANGELOG, version bump
2. Present to user for final approval
3. **STOP and WAIT for human approval before merging to main**

**Stage 7: CURATE**
1. Update all brain files to reflect final state
2. Regenerate state/status training pairs
3. Report memory stats to user
