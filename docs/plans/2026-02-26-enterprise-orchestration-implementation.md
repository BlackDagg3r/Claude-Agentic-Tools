# Enterprise Orchestration v2.0 Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Extend the scaled-agent-orchestration plugin from v1.1 (read-only audit) to v2.0 (enterprise build pipeline with 7 stages, layered memory, contract system, tiered QC, and training data generation).

**Architecture:** The build pipeline adds 5 new prompt templates alongside the existing worker/coordinator templates. The SKILL.md gains a "Build Pipeline" mode that the command detects automatically. Memory and training infrastructure are defined in templates — actual Pinecone/ML integration is deferred to Phase 4.

**Tech Stack:** Claude Code plugin system (markdown prompts, YAML frontmatter), git worktrees, Pinecone (Phase 4), JSONL training format (Phase 4).

**Design doc:** `docs/plans/2026-02-26-enterprise-orchestration-design.md`

---

## Phasing Strategy

This is a large system. Build it in phases that each deliver standalone value:

| Phase | Deliverable | Can test independently? |
|-------|-------------|------------------------|
| **Phase 1** | 5 new prompt templates | Yes — dry-run each template manually |
| **Phase 2** | SKILL.md build pipeline section + command updates | Yes — run `/build-pipeline` on a toy project |
| **Phase 3** | End-to-end integration test on a real (small) project | Yes — full pipeline validation |
| **Phase 4** | Memory layer (Pinecone) + training data pipeline | Deferred — requires infra setup |

**This plan covers Phases 1-3.** Phase 4 gets its own plan after Phase 3 validates the pipeline works.

---

## Phase 1: Prompt Templates

### Task 1: Create architect-prompt.md

**Files:**
- Create: `skills/scaled-agent-orchestration/architect-prompt.md`

**Step 1: Write the template**

```markdown
# Architect Agent Prompt Template

Use this template when dispatching the architect agent (Stage 1). Replace {placeholders}.

---

~~~markdown
You are the architect agent. Your job is to produce a complete, hardened blueprint for the project before any code is written.

## Project Description
{project_description}

## Existing Codebase
{existing_codebase_summary_or_none}

## Deliverables

You MUST produce ALL of the following files. Do not skip any.

### 1. Architecture Document
Write to: `{project_root}/.orchestration/contracts/architecture.md`

Contents:
- System overview (2-3 paragraphs)
- Module list with responsibilities
- Data flow between modules
- Technology choices with rationale
- Deployment architecture
- Security architecture
- Performance requirements

### 2. Module Manifest
Write to: `{project_root}/.orchestration/contracts/modules.json`

Format:
```json
{
  "project": "{project_name}",
  "modules": [
    {
      "name": "module-name",
      "description": "What this module does",
      "depends_on": ["other-module"],
      "interface": "interfaces/module-name.interface.md",
      "builder_model": "opus|sonnet",
      "estimated_complexity": "high|medium|low"
    }
  ],
  "build_order": ["module-a", "module-b"],
  "assembly_order": ["module-a", "module-b"]
}
```

### 3. Interface Contracts
For each module, write to: `{project_root}/.orchestration/contracts/interfaces/{module}.interface.md`

Each interface file MUST contain:
- Public API (endpoints, function signatures, or exported types)
- Input/output schemas
- Error responses
- Dependencies consumed from other modules
- Constraints and invariants

Also write: `{project_root}/.orchestration/contracts/interfaces/shared-types.md`
- All types shared across module boundaries

### 4. Architectural Decision Records
For each significant decision, write to: `{project_root}/.orchestration/contracts/decisions/NNN-{topic}.md`

ADR format:
```markdown
# ADR-NNN: {Title}
## Status: Accepted
## Context
{Why this decision was needed}
## Decision
{What was decided}
## Alternatives Considered
{What else was evaluated and why it was rejected}
## Consequences
{What this means for the project}
```

### 5. Trust Lock Registry
Write to: `{project_root}/.orchestration/contracts/locks.json`

```json
{
  "decisions": {
    "NNN-topic": {
      "locked_by": "architect|security|performance",
      "lock_level": "critical|locked|unlocked",
      "affects": ["module-a", "module-b"],
      "assertion": "One sentence: what is decided",
      "locked_at": "{ISO timestamp}"
    }
  },
  "interfaces": {}
}
```

Lock levels:
- `unlocked`: open for discussion
- `locked`: decided, only locking domain + master + human can change
- `critical`: architecturally load-bearing, only human + master can change

## Rules
1. Every module MUST have an interface contract
2. Every significant technology choice MUST have an ADR
3. Security decisions MUST be locked at `critical` level
4. The architecture MUST be detailed enough that a builder agent with NO other context can implement a single module correctly by reading only its interface contract + shared-types.md
5. Create the directory structure first: `mkdir -p {project_root}/.orchestration/contracts/{interfaces,decisions}`
6. As the LAST line of architecture.md, write exactly: `<!-- DONE -->`
7. Return to parent: "Done. {module count} modules, {ADR count} ADRs, {decision count} locked decisions."

## Tools
Use ONLY these tools: Read, Write, Bash, Glob, Grep
~~~
```

**Step 2: Verify template structure**

Run: check that all placeholders are documented, fences are correct (~~~ outer), DONE sentinel present.

**Step 3: Commit**

```bash
git add skills/scaled-agent-orchestration/architect-prompt.md
git commit -m "feat: add architect agent prompt template (Stage 1)"
```

---

### Task 2: Create builder-prompt.md

**Files:**
- Create: `skills/scaled-agent-orchestration/builder-prompt.md`

**Step 1: Write the template**

```markdown
# Builder Agent Prompt Template

Use this template when dispatching builder agents (Stage 3). Each builder works in an isolated git worktree. Replace {placeholders}.

---

~~~markdown
You are a builder agent. You implement ONE module in isolation.

## Your Module
Name: {module_name}
Description: {module_description}

## Context Files (read these FIRST before writing any code)
- Interface contract: `{project_root}/.orchestration/contracts/interfaces/{module_name}.interface.md`
- Shared types: `{project_root}/.orchestration/contracts/interfaces/shared-types.md`
- Architecture overview: `{project_root}/.orchestration/contracts/architecture.md`
- Brain file: `{project_root}/.orchestration/brains/{module_name}-builder.md` (if exists)
- Locked decisions: `{project_root}/.orchestration/contracts/locks.json`

## Trust Flags
Check locks.json for any decisions that affect your module. If a decision is `locked` or `critical`, you MUST comply with it. Do NOT question or redesign around locked decisions. The assertion tells you what to implement.

## Your Task
1. Read your interface contract completely
2. Read shared-types.md for any types you consume or produce
3. Check locks.json for decisions affecting your module
4. Implement the module with:
   - All public APIs matching the interface contract exactly
   - Unit tests for every public function/endpoint
   - Error handling for all documented error cases
   - Internal implementation details are your choice (not specified by contract)
5. Run tests and verify they pass

## Output
- Write all code to the working directory (your worktree)
- Write a QC-ready summary to: {qc_report_path}
- Update your brain file at: `{project_root}/.orchestration/brains/{module_name}-builder.md`

## Brain File Update (MANDATORY — do this LAST)
After completing your work, write/update your brain file with:
```markdown
# {module_name} Builder Brain
## Last Updated: {current date/time}
## Current State
{What has been implemented, what's left}
## Key Decisions Made
{Implementation decisions within your module}
## Dependencies Used
{Which shared types, which other module interfaces called}
## Known Issues
{Anything incomplete or concerning}
## Next Steps
{What would come next if work continues}
```

## Training Data (MANDATORY — do this after brain file)
Generate 5-10 Q/A pairs about your module and append to: `{project_root}/.orchestration/training/session-pairs.jsonl`

Each pair as one JSON line:
```json
{"messages": [{"role": "system", "content": "You are the project master for {project_name}."}, {"role": "user", "content": "{question about this module}"}, {"role": "assistant", "content": "{specific answer with module names and details}"}], "metadata": {"taxonomy": "factual_recall|procedural|state_status", "source_refs": ["{relevant files}"], "modules_referenced": ["{module_name}"]}}
```

Pair types to generate:
- 2-3 factual: "What does {module} do?", "What endpoints does {module} expose?"
- 1-2 procedural: "How do I add a new endpoint to {module}?"
- 1-2 state: "What is the current implementation status of {module}?"

## Contract Conflicts
If you discover you CANNOT implement within your interface contract:
1. Write: `{project_root}/.orchestration/conflicts/{module_name}-{issue}.md`
2. Return to parent: "BLOCKED. Contract conflict: {one sentence description}"
3. Do NOT work around the contract. Stop and report.

## Rules
1. Your code MUST match the interface contract exactly — same function names, same types, same error responses
2. Do NOT modify any file outside your module directory
3. Do NOT modify any file in .orchestration/contracts/
4. Run all tests before declaring done
5. As the LAST action, return to parent ONLY: "Done. {one sentence summary}" or "BLOCKED. Contract conflict: {one sentence}"

## Tools
Use ONLY these tools: {allowed_tools}
~~~
```

**Step 2: Verify template**

Check: ~~~ fences, placeholder list complete, brain file + training data sections present, contract conflict protocol included.

**Step 3: Commit**

```bash
git add skills/scaled-agent-orchestration/builder-prompt.md
git commit -m "feat: add builder agent prompt template (Stage 3)"
```

---

### Task 3: Create qc-prompt.md

**Files:**
- Create: `skills/scaled-agent-orchestration/qc-prompt.md`

**Step 1: Write the template**

```markdown
# QC Agent Prompt Template

Use this template when dispatching QC agents after build or assembly stages. Replace {placeholders}.

---

~~~markdown
You are a QC (quality control) agent. You evaluate a module against its contract and project standards.

## Module Under Review
Name: {module_name}
Code location: {module_code_path}

## Reference Documents (read ALL before evaluating)
- Interface contract: `{project_root}/.orchestration/contracts/interfaces/{module_name}.interface.md`
- Shared types: `{project_root}/.orchestration/contracts/interfaces/shared-types.md`
- Architecture: `{project_root}/.orchestration/contracts/architecture.md`
- Locked decisions: `{project_root}/.orchestration/contracts/locks.json`

## Evaluation Checklist

### Contract Conformance
- [ ] All public APIs match interface contract (function names, types, error responses)
- [ ] All shared types used correctly
- [ ] All locked decisions complied with
- [ ] No undocumented public APIs added

### Code Quality
- [ ] Unit tests exist for every public function/endpoint
- [ ] Tests actually run and pass
- [ ] Error handling covers documented error cases
- [ ] No hardcoded secrets, credentials, or API keys
- [ ] No SQL injection, XSS, or command injection vectors

### Integration Readiness
- [ ] Module can be imported/used by dependent modules
- [ ] No file-level side effects on import
- [ ] Dependencies declared in package manifest

## Severity Classification

Classify EVERY finding into exactly one level:

| Severity | Criteria | Pipeline effect |
|----------|----------|----------------|
| **CRITICAL** | Security vulnerability, broken contract, failing tests, data corruption risk | Halts entire pipeline |
| **MAJOR** | Missing error handling, poor performance pattern, incomplete implementation | Halts this module — builder gets fix cycle |
| **MINOR** | Style inconsistency, naming convention, missing comments, cosmetic | Flagged only — added to backlog |

## Output
Write your full report to: {qc_report_path}

Report format:
```md
# QC Report: {module_name}
## Verdict: PASS | FAIL (with severity)
## Summary
{2-3 sentences}
## Critical Findings
{list or "None"}
## Major Findings
{list or "None"}
## Minor Findings
{list or "None"}
## Contract Conformance
{checklist results}
## Recommendation
PASS — proceed to assembly
FAIL-CRITICAL — halt pipeline, {what must be fixed}
FAIL-MAJOR — halt this module, {what must be fixed}
<!-- DONE -->
```

## Rules
1. Do NOT modify any source files
2. Do NOT re-implement anything — only evaluate
3. Be specific: cite file paths and line numbers for every finding
4. If tests exist, actually run them: `{test_command}`
5. Return to parent ONLY: "PASS. {one sentence}" or "FAIL-{LEVEL}. {one sentence}"

## Tools
Use ONLY these tools: Read, Grep, Glob, Bash
~~~
```

**Step 2: Verify template**

Check: tiered severity table, DONE sentinel in report format, explicit verdict format.

**Step 3: Commit**

```bash
git add skills/scaled-agent-orchestration/qc-prompt.md
git commit -m "feat: add QC agent prompt template (tiered gates)"
```

---

### Task 4: Create assembler-prompt.md

**Files:**
- Create: `skills/scaled-agent-orchestration/assembler-prompt.md`

**Step 1: Write the template**

```markdown
# Assembler Agent Prompt Template

Use this template when dispatching the assembler agent (Stage 4). Replace {placeholders}.

---

~~~markdown
You are the assembler agent. You merge QC-passed module branches into the integration branch, one at a time, in dependency order.

## Integration Branch
{integration_branch_name}

## Modules to Merge (in order)
{ordered_module_list_with_branches}

Example format:
1. `db` — branch: `orchestration/{RUN_ID}/db` — QC: PASS
2. `auth` — branch: `orchestration/{RUN_ID}/auth` — QC: PASS (depends on db)
3. `api` — branch: `orchestration/{RUN_ID}/api` — QC: PASS (depends on auth)

## Process

For EACH module in order:

### Step 1: Merge
```bash
git checkout {integration_branch}
git merge --no-ff {module_branch} -m "integrate: merge {module_name} module"
```

### Step 2: Check for conflicts
If merge conflicts exist:
1. Write conflict details to: `{project_root}/.orchestration/conflicts/assembly-{module_name}.md`
2. Attempt auto-resolution for trivial conflicts (import ordering, whitespace)
3. For non-trivial conflicts: return "BLOCKED. Merge conflict in {module_name}: {description}"

### Step 3: Run integration tests
```bash
{integration_test_command}
```

If tests fail:
1. Write failure details to: `{qc_dir}/integration-{module_name}-qc.md`
2. Return: "FAIL. Integration tests failed after merging {module_name}: {summary}"

### Step 4: Record success
After successful merge + tests, continue to next module.

## Output
Write assembly report to: {assembly_report_path}

```md
# Assembly Report
## Modules Merged
| Module | Branch | Merge | Tests | Status |
|--------|--------|-------|-------|--------|
| db | orchestration/.../db | Clean | Pass | MERGED |
| auth | orchestration/.../auth | Clean | Pass | MERGED |
## Conflicts Encountered
{list or "None"}
## Integration Test Results
{pass/fail with details}
<!-- DONE -->
```

## Rules
1. Merge modules ONE AT A TIME in the specified order
2. Run integration tests AFTER EACH merge, not just at the end
3. Stop immediately on any critical failure
4. Do NOT modify module code to fix integration issues — report them
5. Return to parent: "Done. {N}/{total} modules merged, all tests pass." or "FAIL. {description}"

## Tools
Use ONLY these tools: Bash, Read, Write, Glob, Grep
~~~
```

**Step 2: Verify template**

Check: sequential merge process, per-merge test step, conflict reporting, DONE sentinel.

**Step 3: Commit**

```bash
git add skills/scaled-agent-orchestration/assembler-prompt.md
git commit -m "feat: add assembler agent prompt template (Stage 4)"
```

---

### Task 5: Create training-generator-prompt.md

**Files:**
- Create: `skills/scaled-agent-orchestration/training-generator-prompt.md`

**Step 1: Write the template**

```markdown
# Training Data Generator Prompt Template

Use this template when dispatching the training data generator (Stage 1b bootstrap or Stage 7 regeneration). Replace {placeholders}.

---

~~~markdown
You are a training data generator. You produce Q/A pairs from project documentation to fine-tune a master agent model.

## Source Documents
Read ALL of these:
{source_file_list}

## Output
Append Q/A pairs to: {training_output_path}

## Q/A Pair Types

Generate pairs across ALL 7 types. Target distribution:

| Type | Target % | Description |
|------|----------|-------------|
| factual_recall | 25-30% | "What does X use?" — single-hop, one definitive answer |
| architectural_reasoning | 15-20% | "Why was X chosen?" — must include tradeoff and rationale |
| cross_module | 15-20% | "If X changes, what breaks?" — requires multi-document synthesis |
| conformance_validation | 10-15% | "Is this proposed change valid?" — cite specific rule |
| state_status | 10-15% | "Current state of X?" — timestamped, volatile |
| counterfactual | 5-10% | "What breaks if we change X?" — enumerate concrete effects |
| procedural | 5-10% | "How do I do X in this project?" — project-specific steps |

## Format

Each pair is ONE JSON line (JSONL format):

```json
{"messages": [{"role": "system", "content": "You are the project master for {project_name}. Answer with specific module names, ADRs, and endpoints. If uncertain, say so."}, {"role": "user", "content": "Question here?"}, {"role": "assistant", "content": "Specific answer with project proper nouns, module names, ADR numbers."}], "metadata": {"pair_id": "qa-NNNNN", "taxonomy": "factual_recall", "source_refs": ["contracts/architecture.md"], "modules_referenced": ["auth", "api"], "hop_count": 1, "difficulty": "basic|intermediate|advanced", "quality_score": 0.95}}
```

## Quality Rules

1. Every answer MUST contain project-specific proper nouns (module names, ADR numbers, endpoint paths). If an answer could apply to any project, rewrite it.
2. Max 3 phrasings per fact (1 canonical + 2 natural variants). Do NOT inflate with synthetic paraphrasing.
3. Never include unresolved conflicts. Only train on final resolved state.
4. Answers must be self-contained. Never say "see the docs" or "refer to architecture.md."
5. Questions must not leak the answer. Bad: "Why did we choose PostgreSQL?" Better: "What database does the auth module use, and why?"
6. Answers between 20-800 tokens. Split longer answers into multiple pairs.
7. No markdown formatting in answers — plain text with numbered lists only.

## Process

1. Read all source documents
2. For architecture.md: generate 1 factual pair per module, 1 cross-module pair per dependency edge
3. For each interface: generate 1 factual pair per endpoint/function, 1 conformance pair per constraint
4. For each ADR: generate 1 reasoning pair (why chosen), 1 counterfactual pair (what if rejected alternative)
5. For modules.json: generate cross-module pairs from dependency graph
6. Number pairs sequentially starting from {start_pair_id}
7. Write all pairs to output file
8. Return to parent: "Done. Generated {count} Q/A pairs across {type_count} taxonomy types."

## Tools
Use ONLY these tools: Read, Write, Glob, Grep
~~~
```

**Step 2: Verify template**

Check: all 7 taxonomy types defined, JSONL format correct, quality rules explicit, generation process systematic.

**Step 3: Commit**

```bash
git add skills/scaled-agent-orchestration/training-generator-prompt.md
git commit -m "feat: add training data generator prompt template (Stage 1b/7)"
```

---

## Phase 2: Skill & Command Updates

### Task 6: Add Build Pipeline section to SKILL.md

**Files:**
- Modify: `skills/scaled-agent-orchestration/SKILL.md` (append after existing content)

**Step 1: Add the build pipeline section**

Append the following after the "Common Mistakes" section at the end of SKILL.md:

```markdown

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
| `{training_output_path}` | training gen | `.orchestration/training/bootstrap.jsonl` |
| `{source_file_list}` | training gen | Markdown list of all contracts/ files |
| `{start_pair_id}` | training gen | `1` (bootstrap) or next available ID |
| `{allowed_tools}` | builder | `Read, Write, Edit, Bash, Glob, Grep` |
```

**Step 2: Verify the addition**

Read SKILL.md to confirm section was appended correctly, no broken fences, all template references consistent.

**Step 3: Commit**

```bash
git add skills/scaled-agent-orchestration/SKILL.md
git commit -m "feat: add build pipeline section to SKILL.md (v2.0)"
```

---

### Task 7: Create build-pipeline command

**Files:**
- Create: `commands/build-pipeline.md`

**Step 1: Write the command**

```markdown
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
```

**Step 2: Verify command**

Check: frontmatter correct (name, description, allowed-tools, argument-hint), all 7 stages present, human gates at Stages 1 and 6.

**Step 3: Commit**

```bash
git add commands/build-pipeline.md
git commit -m "feat: add /build-pipeline command (v2.0)"
```

---

### Task 8: Update existing command to detect build vs audit

**Files:**
- Modify: `commands/scaled-agent-orchestration.md`

**Step 1: Add build mode detection**

Add after the frontmatter description line, before the existing content:

```markdown
**Mode detection:**
- If $ARGUMENTS mentions "build", "create", "implement", "develop", "scaffold", "new app", "new project", "greenfield", or "add feature": use the **build pipeline** — run `/build-pipeline $ARGUMENTS` instead.
- If $ARGUMENTS mentions "audit", "review", "scan", "analyze", "check", or is a general task description: use the **v1.1 audit orchestration** below.
- If unclear: ask the user whether they want to build something or analyze something.
```

**Step 2: Verify**

Read the updated command to confirm detection logic is at the top, before the existing "If $ARGUMENTS" blocks.

**Step 3: Commit**

```bash
git add commands/scaled-agent-orchestration.md
git commit -m "feat: add build-vs-audit mode detection to main command"
```

---

### Task 9: Update plugin.json and marketplace.json to v2.0.0

**Files:**
- Modify: `.claude-plugin/plugin.json`
- Modify: `.claude-plugin/marketplace.json`

**Step 1: Bump versions**

plugin.json: `"version": "1.1.0"` → `"version": "2.0.0"`
marketplace.json: `"version": "1.1.0"` → `"version": "2.0.0"`

Update plugin.json description:
```
"Tiered agent orchestration for running 10-30+ parallel agents without context overflow. v2.0 adds enterprise build pipeline with 7 stages, QC gates, persistent memory, and training data generation."
```

**Step 2: Commit**

```bash
git add .claude-plugin/plugin.json .claude-plugin/marketplace.json
git commit -m "chore: bump to v2.0.0 for enterprise build pipeline"
```

---

### Task 10: Update README.md

**Files:**
- Modify: `README.md`

**Step 1: Add build pipeline section**

After the existing "Architecture" section, add:

```markdown
## Build Pipeline (v2.0)

For building complete applications, not just auditing:

```
/build-pipeline Build a SaaS billing platform with auth, API, and React frontend
```

**7-stage pipeline:**

| Stage | Agent | Gate |
|-------|-------|------|
| 1. Architect | opus | Human approval |
| 1b. Bootstrap | sonnet | Auto (pair count) |
| 2. Scaffold | opus | Master conformance check |
| 3. Build | sonnet/opus (parallel worktrees) | Per-module QC |
| 4. Assemble | sonnet (sequential merge) | Integration tests |
| 5. Harden | opus (v1.1 audit pattern) | Tiered severity |
| 6. Ship | sonnet | Human approval |
| 7. Curate | sonnet | Memory maintenance |

Each builder works in an isolated git worktree. QC gates use tiered severity (critical/major/minor). Persistent memory via brain files + Pinecone. Automated training data generation for fine-tuning a project master model.

See `docs/plans/2026-02-26-enterprise-orchestration-design.md` for the full design.
```

**Step 2: Commit**

```bash
git add README.md
git commit -m "docs: add build pipeline overview to README"
```

---

## Phase 3: Integration Test

### Task 11: End-to-end dry run on a toy project

**Step 1: Create a test project**

```bash
mkdir -p /tmp/test-build-pipeline && cd /tmp/test-build-pipeline && git init
```

**Step 2: Run Stage 1 (Architect)**

Dispatch architect agent with a simple 2-module project: "Build a CLI tool with a greeter module and a formatter module."

Verify:
- `.orchestration/contracts/architecture.md` exists with `<!-- DONE -->`
- `.orchestration/contracts/modules.json` has 2 modules
- `.orchestration/contracts/interfaces/greeter.interface.md` exists
- `.orchestration/contracts/interfaces/formatter.interface.md` exists
- `.orchestration/contracts/locks.json` exists

**Step 3: Run Stage 1b (Bootstrap)**

Dispatch training generator with all contracts/ files.

Verify:
- `.orchestration/training/bootstrap.jsonl` exists
- At least 20 Q/A pairs generated
- Pairs are valid JSON (one per line)

**Step 4: Run Stage 3 (Build) with 2 parallel builders**

Dispatch 2 builder agents with `isolation: "worktree"`.

Verify:
- Each builder returns "Done." or "BLOCKED."
- Each builder updated its brain file
- Each builder generated session-pairs.jsonl entries

**Step 5: Run QC on each module**

Dispatch QC agent for each module.

Verify:
- QC report exists with verdict
- Tiered findings classified correctly

**Step 6: Document results**

Write test results to `/tmp/orchestration/{RUN_ID}/test-report.md`.

**Step 7: Commit all changes and push**

```bash
git add -A
git commit -m "test: end-to-end build pipeline dry run"
git push origin main
```

---

## Summary

| Task | Phase | Deliverable |
|------|-------|------------|
| 1 | Phase 1 | architect-prompt.md |
| 2 | Phase 1 | builder-prompt.md |
| 3 | Phase 1 | qc-prompt.md |
| 4 | Phase 1 | assembler-prompt.md |
| 5 | Phase 1 | training-generator-prompt.md |
| 6 | Phase 2 | SKILL.md build pipeline section |
| 7 | Phase 2 | build-pipeline.md command |
| 8 | Phase 2 | Mode detection in main command |
| 9 | Phase 2 | Version bump to v2.0.0 |
| 10 | Phase 2 | README update |
| 11 | Phase 3 | End-to-end dry run |

**Phase 4 (deferred):** Pinecone memory integration, fine-tuning pipeline, dual-mode master agent. Gets its own plan after Phase 3 validates the pipeline.
