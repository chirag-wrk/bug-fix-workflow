# openspec-bugfix-workflow

Custom [OpenSpec](https://github.com/Fission-AI/OpenSpec) schema for **gated, Jira-driven bug fixing** with AI-assisted triage, reproduction, RCA, planning, and implementation. Supports two code-generation strategies: **ai-helpers** (OAPE command routing + eval gate) and **direct** (plain agent implementation).

---

## Quick Start

### 1. Clone & Install

```bash
rm -rf /tmp/openspec-workflow
git clone -b openspec-operator-generic https://github.com/sujkini/openspec.git /tmp/openspec-workflow
/tmp/openspec-workflow/install.sh /path/to/your-operator-repo
```

This copies `openspec/`, `.cursor/`, `eval-generation/`, and `dashboard/` into your project, installs the OpenSpec CLI, and sets up dependencies. Use `--no-dashboard` to skip the dashboard.

### 2. Choose code generation mode (`openspec/config.yaml`)

Set `flags.codegen_mode` before you start implementing tasks:

```yaml
# openspec/config.yaml
flags:
  codegen_mode: ai-helpers   # or: direct
```

| Mode | When to use | What `/opsx-apply` does |
|------|-------------|-------------------------|
| **`ai-helpers`** (default) | API/controller/e2e work that benefits from specialized OAPE Cursor commands and a code eval gate | design-bundle → OAPE command → verify → code eval → refine → approve |
| **`direct`** | Straightforward fix tasks; simpler/faster path | agent reads context → FILE OPERATIONS → verify → approve (no OAPE, no code eval) |

Change the flag anytime; `/opsx-apply` reads it on each invocation. Details below under [Configuration](#configuration-openspecconfigyaml).

### 3. Start the Dashboard (optional)

```bash
cd /path/to/your-operator-repo
./dashboard/start.sh
```

Installs deps on first run, starts the FastAPI backend (port 8000) and React frontend (port 5173). Open http://localhost:5173. See `dashboard/README.md` for details.

### 4. Restart Cursor

Restart Cursor so slash commands load from `.cursor/commands/`.

### 5. Run your first bug-fix change

Open the **operator repo** as the Cursor workspace (recommended for working-folder mode), then:

```
/opsx-new PROJ-123
```

Only the **Jira Bug ticket key** is required at `/opsx-new`. Then progress with `/opsx-continue` through:

```
bug-validation.json → bug-report.md → repro-verification-report.md → rca-report.md
  → bugfix-plan.md → tasks.md → /opsx-apply → archive
```

**Repro Verification tip (OpenShift operators):** Prefer reproducing in Cursor agent chat via repo-local tests/envtest, make targets, or must-gather / user-provided logs. Use a live cluster only when access exists and repo-local evidence is insufficient. If a real OpenShift cluster is required but unavailable, mark **Partial** and document the limitation — do not invent live-cluster output. See [Repro Verification](#repro-verification).

---

## Configuration

After installation, configure two files in `openspec/inputs/`:

| File | What to define |
|------|---------------|
| **`openspec/inputs/agents.md`** | Agent routing, repository architecture, test patterns, verification matrix |
| **`openspec/inputs/constitution.md`** | Coding guardrails, CI gates, governance rules |

These are the **only operator-specific files**. Everything else is generic.

Your `agents.md` should define:
- **Repository layout** — directory structure, key packages
- **Architecture patterns** — controller frameworks, reconciliation flow
- **Test exemplar** — how tests are structured (mocks, table-driven patterns, file naming)
- **Execution agent routing** — agent IDs and which paths/packages they own
- **Per-task verification matrix** — `make` targets and `go test` commands per task type

The bundled `agents.md` ships with a reference. Replace it entirely with your operator's documentation.

---

## Running the Workflow

### Start a change

```
/opsx-new PROJ-123
```

Primary input: **Jira Bug ticket key** (stored in `inputs/jira.yaml`). Target GitHub repo is required before **Repro Verification** (unless working-folder mode).

### Progress through artifacts

```
/opsx-continue              → bug-validation.json           [approve]
/opsx-continue              → bug-report.md                 [approve]
/opsx-continue              → repro-verification-report.md  [approve]
/opsx-continue              → rca-report.md                 [approve]
/opsx-continue              → bugfix-plan.md               [approve]
/opsx-continue              → tasks.md                      [approve]
```

Each artifact is:
1. Generated from the template
2. Evaluated against stage evals
3. Refined if needed
4. Presented for your approval

If you **reject**, the agent refines and re-runs evals until you approve (except `bug-report.md`, which exits on reject per config). Previously approved artifacts stay immutable.

### Implement tasks

```
/opsx-apply                 → task T1 [approve] → task T2 [approve] → … → done
```

The implementation flow depends on `codegen_mode` in `openspec/config.yaml`:

**ai-helpers mode** (`codegen_mode: ai-helpers`):
1. Compose `design-bundle.md` scoped to that task
2. Resolve one OAPE command (or manual work)
3. Run in fork working copy (or project cwd in working-folder mode)
4. Verify against acceptance criteria (including regression / repro checks)
5. Run code-generation evals → refine code (max 2 passes)
6. Present task summary + scorecard → user approval
7. On approve: mark task complete, next task

**direct mode** (`codegen_mode: direct`):
1. Read context files (agents.md, constitution.md, bug report, RCA, plan)
2. Implement code directly via FILE OPERATIONS
3. Verify against acceptance criteria
4. Present task summary → user approval
5. On approve: mark task complete, next task

### Archive

```
/opsx-archive               → archive the change
```

---

## Repro Verification

Repro Verification confirms the bug is reproducible and captures a failure signature for RCA. It runs in Cursor agent chat against the operator repo when possible.

| Execution mode | When to use | Log source to record |
|----------------|-------------|----------------------|
| **Repo-local / Cursor chat** | Unit/integration tests, envtest, make targets, or code-path checks confirm the signature | `repo-local tests/envtest` |
| **Must-gather / user logs** | No live cluster; logs in workspace match the reported failure | `must-gather` or `user-provided logs` |
| **Live cluster** | kubeconfig/cluster access available and repo-local evidence is insufficient | `live cluster` |
| **Partial** | Real OpenShift cluster (or other env) is required but unavailable | Document gap in Assessment Limitations |

**Practical rule:** If the bug reproduces via tests/envtest or is confirmed from must-gather, complete the whole Repro Verification stage in Cursor chat. If it needs a real OpenShift cluster and you do not have access, mark **Partial** and document that limitation — do not pretend live steps ran.

Provide must-gather, operator logs, or cluster access before `/opsx-continue` for repro when the bug cannot be confirmed from the repo alone.

---

## Working Modes

### Mode A: Working-folder mode (local code changes)

Use when your Cursor workspace IS the operator repo.

When prompted for target repo, tell the agent: **"use this as the working directory"**
- Code changes happen directly in your working directory
- No fork URL needed, no draft PR
- Ideal for repo-local repro (tests/envtest) in the same chat

### Mode B: Fork mode (draft PR)

When prompted, provide:
- **Target repo URL** — before Repro Verification
- **Fork repo URL** — before `/opsx-apply`

The agent clones your fork, implements task-by-task, and opens a draft PR.

---

## Cursor Commands

### Forward workflow

| Command | Purpose |
|---------|---------|
| `/opsx-new PROJ-123` | Start a bug-fix change from a Jira Bug key |
| `/opsx-continue` | Create next artifact; eval gate; approval |
| `/opsx-apply` | Implement tasks — one at a time, approval after each |
| `/opsx-archive` | Archive a completed change |
| `/opsx-explore` | Explore ideas without creating artifacts |

### OAPE commands (ai-helpers mode only, during `/opsx-apply`)

| Command | When |
|---------|------|
| `/oape:api-generate` | API_Agent task |
| `/oape:api-generate-tests` | API_Agent verification task |
| `/oape:api-implement` | OperatorController_Agent task |
| `/oape:e2e-generate` | E2E / Testing_Agent task |

These commands are **not used** when `codegen_mode: direct`.

### Retrospective eval loop

| Command | Purpose |
|---------|---------|
| `/eval-loop` | Improve evals from a completed feature/bugfix bundle |

---

## Configuration (`openspec/config.yaml`)

Key flags you can tune:

```yaml
flags:
  codegen_mode: ai-helpers       # "ai-helpers" or "direct"
  max_feedback_rounds: 3
  exit_on_all_tasks_complete: true
```

| Flag | Default | What it does |
|------|---------|--------------|
| `codegen_mode` | `ai-helpers` | Code generation strategy: `ai-helpers` (OAPE commands + code eval gate) or `direct` (plain agent, no OAPE, no eval gate) |
| `max_feedback_rounds` | 3 | Max rejection + refinement loops per artifact before halting |
| `exit_on_all_tasks_complete` | true | Auto-exit implementation when all tasks marked `[x]` |

### Code generation modes

**`ai-helpers`** — For each task, composes a `design-bundle.md`, routes to specialized OAPE Cursor commands (`api-generate`, `api-implement`, `e2e-generate`), scores generated code via a code-generation eval gate, refines until evals pass, then asks for user approval.

**`direct`** — The Cursor agent reads context files directly, implements code via FILE OPERATIONS, verifies against acceptance criteria, and asks for user approval. No OAPE commands, no design bundles, no code eval gate. Simpler and faster for straightforward fix tasks.

---

## Eval Loop (Optional, Recommended)

The eval loop is a **retrospective improvement** tool. After a bug fix (or feature) is fully completed, feed its history into `/eval-loop` to generate eval cases that improve the quality of future runs.

### Step 1: Provide inputs

Fill `eval-generation/input/feature-bundle.yaml` with data from a **completed change**:

| Field | What to paste |
|-------|---------------|
| `feature_name` | Change / bug name |
| `epic_key` | Jira epic key |
| `target_repo` | Target repository URL |
| `enhancement_proposal` | Full EP/ARD content |
| `jira_epic` | Jira epic export |
| `repo_state` | Pre-change repo state |
| `user_stories` | User stories linked to the epic |
| `repo_prs` | PR links and key diffs |
| `bugs` | Bug list with root causes |

### Step 2: Run the eval loop

```
/eval-loop
```

### Step 3: Review template gaps

Review the gap reports generated in:

```
eval-generation/eval-generation-workflow/template-gaps/
```

### Step 4: Review refined templates

Find refined templates in:

```
eval-generation/output-refined-templates/
```

### Step 5: Apply approved refinements

If you approve the refined templates, copy them into the active workflow:

```bash
cp eval-generation/output-refined-templates/*.md openspec/schemas/openspec-bugfix-workflow/templates/
```

### Step 6: Evals are auto-synced

The generated evals in `eval-generation/output-evals/` are automatically copied to:

```
openspec/schemas/openspec-bugfix-workflow/evals/
```

These evals run as quality gates during `/opsx-continue` for every future artifact.

### Repeating

Update `eval-generation/input/feature-bundle.yaml` with the next completed change and run `/eval-loop` again. Prior evals accumulate — each round improves coverage.

---

## Pipeline Overview

```
bug-validation → bug-report → repro-verification → rca → bugfix-plan → tasks → implementation → archive
```

| Stage | Artifacts | Purpose |
|-------|-----------|---------|
| **Bug Triage** | `bug-validation.json`, `bug-report.md` | Validate Jira bug completeness; structure bug context, ARD, and original PRs |
| **Repro Verification** | `repro-verification-report.md` | Confirm reproducibility (repo-local tests/envtest, must-gather, or live cluster); capture failure signature |
| **Root Cause Analysis** | `rca-report.md` | Trace failure path from signature to root cause |
| **Bug Fix Planning** | `bugfix-plan.md` | Single-phase fix plan, regression strategy, rollback |
| **Constitution (input)** | `constitution.md` (resolved) | Non-negotiable guardrails (resolved before planning) |
| **Task creation** | `tasks.md` | Executable backlog: code fix, regression test, verification |
| **Implementation** | code + `implementation-report.md` | Task-by-task execution with per-task approval (ai-helpers or direct mode) |
| **Archive** | archived change | Close out |

---

## Prerequisites

| Requirement | Notes |
|-------------|-------|
| [Node.js](https://nodejs.org/) | For OpenSpec CLI installation |
| [OpenSpec CLI](https://github.com/Fission-AI/OpenSpec) | Installed by `install.sh` |
| [Cursor](https://cursor.com) | Slash commands load from `.cursor/commands/` |
| Jira access | Bug ticket key at `/opsx-new`; content via MCP or paste |
| Target GitHub repo | URL before **Repro Verification**; or use working-folder mode |
| Fork GitHub repo | URL before `/opsx-apply`; skip in working-folder mode |
| Logs / cluster (conditional) | Must-gather, user logs, or cluster access when repo-local repro is insufficient |

---

## Repository Layout

```
.
├── openspec/                              # Pre-built — ready to use after install
│   ├── config.yaml                        # Workflow configuration and flags
│   ├── inputs/                            # Operator-specific inputs (edit these)
│   │   ├── agents.md                      # Agent routing, architecture, test patterns
│   │   └── constitution.md                # Coding guardrails, CI gates, governance
│   ├── schemas/openspec-bugfix-workflow/ # Schema, templates, stage-gate, evals
│   │   ├── schema.yaml                    # Workflow definition
│   │   ├── templates/                     # Generic artifact templates (*-template.md)
│   │   ├── evals/                         # Stage eval cases (quality gates)
│   │   ├── stage-gate/                    # Eval gate prompts and artifact map
│   │   └── feedback_stage_artifacts/      # Format spec for rejection rounds
│   └── changes/                           # Active changes (created per /opsx-new)
├── .cursor/                               # Pre-built — Cursor loads immediately
│   ├── commands/                          # opsx-new, opsx-continue, opsx-apply, eval-loop
│   └── skills/                            # openspec-*, effective-go, e2e-test-generator
├── eval-generation/                       # Retrospective eval loop
│   ├── input/                             # feature-bundle.yaml (your input)
│   ├── output-evals/                      # Generated evals per stage (auto-synced)
│   ├── output-refined-templates/          # Refined templates (review before applying)
│   └── eval-generation-workflow/          # Internal workflow machinery
│       ├── template-gaps/                 # Gap reports per template
│       ├── outputs/                       # Epic-bug-analysis + patches
│       ├── rounds/                        # Round snapshots
│       └── generation-phase/              # SYSTEM_PROMPT, template-inventory
├── dashboard/                             # Observability dashboard (optional)
│   ├── config.json                        # Dashboard configuration
│   ├── start.sh                           # One-command launcher
│   ├── src/                               # FastAPI backend (ingest + UI)
│   └── web/                               # React + TypeScript SPA
├── install.sh                             # Installer script
└── README.md
```

---

## agents.md Resolution (lookup order)

1. `{target_repo}/AGENTS.md`
2. `{target_repo}/agents.md`
3. `openspec/inputs/agents.md`
4. `{schema_root}/agents.md` (bundled fallback)

## constitution.md Resolution (lookup order)

1. `{target_repo}/constitution.md`
2. `{target_repo}/CONSTITUTION.md`
3. `openspec/inputs/constitution.md`

If not found, the agent generates one using `templates/constitution-template.md`.

---

## Validate Schema

```bash
openspec schema validate openspec-bugfix-workflow
```

---

## License

MIT (schema and templates). OpenSpec CLI is separate — see [Fission-AI/OpenSpec](https://github.com/Fission-AI/OpenSpec).
