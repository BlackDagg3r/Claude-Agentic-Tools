# Enterprise Agent Orchestration — Design Document

**Date:** 2026-02-26
**Status:** Approved
**Authors:** atwellguy, Claude Opus 4.6

---

## 1. Overview

An extension to the scaled-agent-orchestration plugin (v1.1) that handles building complete enterprise applications from planning through shipping and maintenance. The system uses a 7-stage pipeline with isolated parallel builders, tiered QC gates, layered persistent memory, a trust flag system for cross-domain decisions, and automated training data generation to fine-tune a local master agent that knows the project intimately.

**Core principles:**

- Plan hard, build fast, never re-learn
- Agents write code in isolation (git worktrees), assemble sequentially
- Blueprints are sacred — only human + master + domain expert can modify
- Every agent session produces memory artifacts AND training data
- The master agent gets smarter at every stage

**Compatibility:** The existing v1.1 skill works unchanged for read-only audit/analysis tasks. The build pipeline is an additional mode, not a replacement.

| User request | Mode | Pipeline |
|-------------|------|----------|
| "Audit this codebase for security issues" | v1.1 audit | Workers → Coordinators → Summary |
| "Build a SaaS billing platform" | v2.0 build | Architect → Scaffold → Build → Assemble → Harden → Ship |
| "Add RBAC to existing app" | v2.0 build | Architect (scoped) → Build (new modules) → Assemble → Harden |

---

## 2. The Pipeline

### Stage 1: ARCHITECT (opus, foreground, human approves)

- **Inputs:** Human description, existing codebase (if any)
- **Agents:** 1 architect agent (opus)
- **Produces:**
  - `.orchestration/contracts/architecture.md`
  - `.orchestration/contracts/modules.json`
  - `.orchestration/contracts/interfaces/{module}.interface.md`
  - `.orchestration/contracts/decisions/NNN-{topic}.md` (ADRs)
  - `.orchestration/contracts/locks.json`
- **Gate:** HUMAN APPROVAL (reviews full blueprint)

### Stage 1b: BOOTSTRAP MASTER

- **Inputs:** All contracts/ produced by Stage 1
- **Agents:** 1 training-data generator (sonnet)
- **Produces:** `.orchestration/training/bootstrap.jsonl` (~200-500 Q/A pairs)
- **Action:** Fine-tune base model, deploy to LM Studio as master
- **Gate:** Automated (verify pair count > 200, quality > 0.7)

### Stage 2: SCAFFOLD (opus, foreground, sequential)

- **Inputs:** contracts/, master agent available
- **Agents:** 1 scaffold agent (opus)
- **Produces:** Base repo structure, shared types, interfaces, CI config, package.json/pyproject.toml, DB schema stubs, test harness, linter config
- **Gate:** Master agent verifies scaffold matches contracts (conformance validation)

### Stage 3: BUILD (sonnet/opus, background, PARALLEL in worktrees)

- **Inputs:** Scaffolded repo, contracts/, brain files
- **Agents:** 1 builder per module, each in its own git worktree (wave batched: opus 4, sonnet 8, haiku 12)
- **Each builder:**
  1. Gets isolated worktree (`isolation: "worktree"`)
  2. Reads L1 brain file + L2 session summaries
  3. Reads its module interface contract
  4. Implements module + unit tests
  5. Self-curates: updates brain file, writes session summary, generates 5-10 Q/A pairs
  6. Returns: "Done. {summary}" or "BLOCKED. Contract conflict: {description}"
- **Produces:** Each module on its own branch: `orchestration/{RUN_ID}/{module}`
- **Gate:** Per-module QC (tiered — see Section 6)

### Stage 4: ASSEMBLE (sonnet, foreground, SEQUENTIAL)

- **Inputs:** QC-passed module branches, integration branch
- **Agents:** 1 assembler agent (sonnet)
- **Process:** For each passed module (ordered by dependency):
  1. Merge module branch into integration branch
  2. Run integration tests
  3. If conflict: master agent resolves or flags human
  4. If tests fail: route back to builder for fix cycle
- **Produces:** Clean integration branch with all modules merged, cross-module Q/A pairs
- **Gate:** All merges clean, integration tests pass

### Stage 5: HARDEN (opus, parallel — uses v1.1 fan-out pattern)

- **Inputs:** Integrated codebase on integration branch
- **Agents:** Read-only workers + coordinators (existing v1.1 skill)
- **Domains:**
  - Security: OWASP audit, dependency scan, secrets check
  - Performance: profiling, query analysis, memory patterns
  - Edge cases: error paths, boundary conditions, concurrency
  - Compliance: licensing, accessibility, API standards
- **Produces:** Coordinator summaries + conformance/counterfactual Q/A pairs
- **Gate:** Tiered — critical stops pipeline, major gets fix cycle, minor flagged for backlog

### Stage 6: SHIP (sonnet, foreground)

- **Inputs:** Hardened integration branch
- **Agents:** 1 shipping agent (sonnet)
- **Produces:** Final CI pass, generated docs, CHANGELOG, version bump, deployment config
- **Gate:** HUMAN APPROVAL before merge to main

### Stage 7: CURATE (sonnet, background)

Runs after every work session (end of day or end of stage):

1. Compress L2 session summaries → L3 deep archive
2. Update all L1 brain files to reflect final state
3. Purge expired Q/A pairs, regenerate state/status pairs
4. Run quality checks on accumulated training data
5. If training threshold reached: trigger fine-tune cycle
6. Flag contradictions or unresolved conflicts for human review

---

## 3. Memory Architecture

### Layer Definitions

**L1 — Brain Files (git, always injected)**

- Location: `.orchestration/brains/{role}.md`
- Size: ~2,000 words per role
- Updated by: the producing agent at end of each session
- Contains: current module state, active decisions, next steps
- Injected: always, at agent spawn (zero query cost)

**L2 — Session Summaries (Pinecone, recent)**

- Namespace: `project-{name}/{role}`
- Retention: last 5-10 sessions per role
- Written by: producing agent during self-curation
- Contains: what was done, what was decided, what changed, trust flags
- Retrieved: automatically at agent spawn (cheap query)

**L3 — Deep Archive (Pinecone, historical)**

- Namespace: `project-{name}/archive`
- Retention: indefinite
- Compressed from: L2 by Stage 7 curator
- Contains: full rationale, resolved debates, old decisions
- Retrieved: on-demand semantic query only

**L4 — Training Dataset (JSONL files, accumulating)**

- Location: `.orchestration/training/`
- Format: chat-completion JSONL with metadata
- Feeds: periodic fine-tuning of local master model
- 7 pair types: factual, reasoning, cross-module, conformance, state, counterfactual, procedural
- Target: 2,000-3,000 pairs for production-grade master

### Agent Session Lifecycle

Every agent follows this lifecycle, regardless of role:

```
1. LOAD    → Read L1 brain file (always)
             Retrieve L2 recent summaries (automatic)
             Query L3 if specific deep context needed
2. WORK    → Execute assigned task
3. OUTPUT  → Write deliverables (code, analysis, etc.)
4. CURATE  → Update own L1 brain file
             Write L2 session summary with trust flags
             Generate 5-10 Q/A pairs for L4
             Prune redundant entries from today's session
5. RETURN  → Brief summary to parent
```

Self-curation is performed by the producing agent — the agent that did the work understands best what matters and what's noise. A separate end-of-day curator (Stage 7) handles cross-role consistency and L2→L3 compression.

---

## 4. Contract System & Blueprint Protection

### Directory Structure

```
.orchestration/contracts/
├── architecture.md           System-level design (THE master document)
├── modules.json              Machine-parseable module definitions
├── interfaces/
│   ├── {module}.interface.md One per module — public API contract
│   └── shared-types.md       Types shared across module boundaries
├── decisions/
│   ├── NNN-{topic}.md        Architectural Decision Records
│   └── ...
└── locks.json                Trust flag registry
```

### modules.json Schema

```json
{
  "project": "project-name",
  "modules": [
    {
      "name": "auth",
      "description": "Authentication and authorization",
      "depends_on": ["db"],
      "interface": "interfaces/auth.interface.md",
      "builder_model": "opus",
      "estimated_complexity": "high",
      "worktree_branch": "orchestration/{RUN_ID}/auth"
    }
  ],
  "build_order": ["db", "auth", "api", "frontend"],
  "assembly_order": ["db", "auth", "api", "frontend"]
}
```

### Modification Rules

| Asset | Who can modify | Process |
|-------|---------------|---------|
| `architecture.md` | Human + master only | Human approves all changes |
| `modules.json` | Human + master only | Must match architecture.md |
| Interface files | Human + master + domain expert | Locking domain approves |
| Decision ADRs | Human + master + locking domain | New ADR supersedes old |
| `locks.json` | Human + master only | Audit trail required |
| Builder code | Builder agent in own worktree | Must conform to contracts |

### Contract Conflict Protocol

When a builder can't implement within its contract:

1. Builder writes: `.orchestration/conflicts/{module}-{issue}.md`
2. Builder returns: "BLOCKED. Contract conflict: {one sentence}"
3. Pipeline pauses THAT module only (others continue)
4. Master agent evaluates: flex contract or adapt builder?
5. Human approves resolution
6. If contract changes: all affected builders notified via brain files
7. Resolved conflict becomes a new ADR

---

## 5. Trust Flag System

### locks.json

```json
{
  "decisions": {
    "001-auth-jwt": {
      "locked_by": "security",
      "lock_level": "critical",
      "affects": ["auth", "api", "frontend"],
      "assertion": "JWT with RS256, 15-min TTL, refresh rotation",
      "locked_at": "2026-02-26T10:30:00Z"
    }
  },
  "interfaces": {
    "auth.interface.md": {
      "locked_by": "architect",
      "lock_level": "locked",
      "last_modified_by": "human"
    }
  }
}
```

### Lock Levels

| Lock Level | Meaning | Who can modify |
|-----------|---------|---------------|
| `unlocked` | Open for discussion | Any domain agent |
| `locked` | Decided — don't revisit unless blocked | Locking domain + master + human |
| `critical` | Architecturally load-bearing | Human + master only |

Agents see lock flags (e.g., `[SECURITY-CRITICAL]`) on decisions and respect them without needing the full rationale. The assertion provides enough context to design around the constraint. If an agent must understand why, it queries L3 for the full analysis.

---

## 6. QC Gates (Tiered)

### Severity Levels

| Severity | Examples | Pipeline effect |
|----------|---------|----------------|
| **Critical** | Security vulnerability, data corruption risk, broken contract, failing tests | Halts entire pipeline |
| **Major** | Missing error handling, poor performance pattern, incomplete implementation | Halts that module only — builder gets fix cycle |
| **Minor** | Style inconsistency, naming convention, missing comments | Flagged only — added to backlog |

### QC Checkpoints

| After stage | QC type | Model | Scope |
|-------------|---------|-------|-------|
| Stage 2 (Scaffold) | Conformance | Master agent | Does scaffold match contracts? |
| Stage 3 (Build) | Per-module | sonnet per module | Tests pass? Contract honored? No critical issues? |
| Stage 4 (Assemble) | Integration | sonnet | Cross-module tests pass? Merge conflicts resolved? |
| Stage 5 (Harden) | Full audit | opus (v1.1 fan-out) | Security, performance, edge cases, compliance |
| Stage 6 (Ship) | Final | Human | Approve for merge to main |

---

## 7. Training Data Generation

### Q/A Pair Taxonomy

| Type | Target % | Purpose | Generated from |
|------|----------|---------|---------------|
| Factual recall | 25-30% | "What does X use?" | Architecture docs, contracts, schemas |
| Architectural reasoning | 15-20% | "Why was X chosen?" | ADRs, decision records |
| Cross-module | 15-20% | "If X changes, what breaks?" | Interface contracts, dependency graph |
| Conformance validation | 10-15% | "Is this proposed change valid?" | Contracts + code review findings |
| State/status | 10-15% | "Current state of X?" | Session summaries, build reports |
| Counterfactual | 5-10% | "What breaks if we change X?" | ADRs (rejected alternatives) |
| Procedural | 5-10% | "How do I do X in this project?" | Project-specific workflows |

### Generation Schedule

| Pipeline stage | Pairs generated | Type |
|----------------|----------------|------|
| Stage 1 (Architect) | ~200-500 bootstrap | Factual, reasoning, cross-module |
| Stage 1b (Bootstrap) | Fine-tune triggered | — |
| Stage 3 (Build) | 5-10 per builder session | Factual, procedural, state |
| Stage 4 (Assemble) | 10-20 from integration | Cross-module, conformance |
| Stage 5 (Harden) | 20-40 from audit findings | Conformance, counterfactual |
| Stage 7 (Curate) | Regenerate state pairs, purge expired | State, all types refreshed |

### Quality Gates

- Quality score < 0.7 → excluded from training
- Cosine similarity > 0.92 to existing pair → deduplicated
- Answer could apply to any project → rejected (must contain project-specific proper nouns)
- Max 3 phrasings per fact (1 canonical + 2 natural variants)
- Never train on unresolved conflicts — final resolved state only
- Expiry enforced: state pairs 14 days, interface contracts 60 days, architecture 90 days

### Volume Targets

- Bootstrap (after planning): ~500 pairs
- Effective minimum: 1,000-1,500 pairs
- Production-grade: 2,000-3,000 pairs
- Diminishing returns: beyond 5,000

### JSONL Format

```json
{
  "messages": [
    {
      "role": "system",
      "content": "You are the project master for [PROJECT]. Answer with specific module names, ADRs, and endpoints."
    },
    {
      "role": "user",
      "content": "What auth strategy does the project use?"
    },
    {
      "role": "assistant",
      "content": "JWT with RS256 signing, 15-min access tokens, refresh rotation. Defined in ADR-001, implemented in auth module. [SECURITY-CRITICAL]"
    }
  ],
  "metadata": {
    "pair_id": "qa-00001",
    "taxonomy": "factual_recall",
    "source_refs": ["decisions/001-auth-jwt.md"],
    "source_timestamp": "2026-02-26T10:30:00Z",
    "generated_at": "2026-02-26T11:00:00Z",
    "difficulty": "basic",
    "hop_count": 1,
    "modules_referenced": ["auth"],
    "expires_at": "2026-05-26T10:30:00Z",
    "quality_score": 0.95
  }
}
```

---

## 8. Master Agent (Dual Mode)

| Mode | Model | Used for | Context source |
|------|-------|----------|---------------|
| **Cloud master** | Claude opus | Architecture decisions, novel problems, complex reasoning, conflict resolution | L1 + L2 + L3 (Pinecone RAG) |
| **Local master** | Fine-tuned model (LM Studio) | Project knowledge queries, conformance checks, onboarding, consistency validation | Knowledge baked into weights |

The local master handles high-volume, low-reasoning queries. The cloud master handles low-volume, high-reasoning work. Together they provide a master agent that knows the project intimately without context window limitations.

---

## 9. Directory Structures

### Orchestration Run Directory

```
/tmp/orchestration/{RUN_ID}/
├── workers/              (v1.1 — audit/analysis output)
├── synthesis/            (v1.1 — coordinator summaries)
├── pipeline.md           (stage progression tracker)
├── builds/               (git worktrees for parallel builders)
│   ├── module-auth/      (worktree → branch orchestration/{RUN_ID}/auth)
│   ├── module-api/       (worktree → branch orchestration/{RUN_ID}/api)
│   └── module-frontend/  (worktree → branch orchestration/{RUN_ID}/frontend)
├── qc/                   (QC reports per module per stage)
│   ├── auth-build-qc.md
│   ├── api-build-qc.md
│   └── integration-qc.md
└── conflicts/            (contract conflict reports)
```

### Project Repo

```
.orchestration/
├── contracts/
│   ├── architecture.md
│   ├── modules.json
│   ├── interfaces/
│   ├── decisions/
│   └── locks.json
├── brains/
│   ├── master.md
│   ├── auth-builder.md
│   ├── api-builder.md
│   └── frontend-builder.md
├── training/
│   ├── bootstrap.jsonl
│   ├── session-pairs.jsonl
│   ├── audit-pairs.jsonl
│   └── manifest.json
└── config.json
```

---

## 10. Plugin File Structure

```
scaled-agent-orchestration/
├── .claude-plugin/
│   ├── plugin.json                    (v2.0.0)
│   └── marketplace.json
├── commands/
│   ├── scaled-agent-orchestration.md  (updated — detect build vs audit)
│   └── build-pipeline.md             (NEW — dedicated build command)
├── skills/
│   └── scaled-agent-orchestration/
│       ├── SKILL.md                   (updated — add build pipeline section)
│       ├── worker-prompt.md           (existing v1.1)
│       ├── coordinator-prompt.md      (existing v1.1)
│       ├── builder-prompt.md          (NEW)
│       ├── qc-prompt.md              (NEW)
│       ├── assembler-prompt.md        (NEW)
│       ├── architect-prompt.md        (NEW)
│       └── training-generator-prompt.md (NEW)
├── README.md
└── LICENSE
```

### New Prompt Templates

- `architect-prompt.md` — produces contracts/, modules.json, ADRs
- `builder-prompt.md` — implements one module in a worktree, self-curates, generates Q/A pairs
- `qc-prompt.md` — evaluates a module against contracts, classifies findings as critical/major/minor
- `assembler-prompt.md` — merges branches sequentially, runs integration tests
- `training-generator-prompt.md` — generates Q/A pairs from source documents

---

## 11. Design Decisions and Open Questions

### Decided

- Hybrid human-in-loop: approve architecture, autonomous build, review before ship
- Both JSON manifest + markdown spec for contracts
- Tiered QC gates: critical/major/minor
- Git worktrees for parallel builder isolation
- Layered memory: L1 brain files, L2 session summaries, L3 deep archive (Pinecone)
- Self-curation by producing agent (domain expert cleans own memory)
- Trust flags with lock levels instead of full rationale propagation
- Training data generated as byproduct of every stage
- Bootstrap master fine-tune immediately after planning
- Dual-mode master: cloud (opus) for reasoning, local (fine-tuned) for knowledge

### Open — Defer to Specialists

- Optimal data management for Pinecone namespaces (consult data scientists)
- Fine-tuning hyperparameters and base model selection (consult ML engineers)
- Embedding model choice for Q/A quality scoring (consult data scientists)
- L2→L3 compression algorithm specifics (consult data engineers)
