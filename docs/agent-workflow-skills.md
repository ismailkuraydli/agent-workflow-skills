# Agent Workflow Skills

**Created:** 2026-09-21
**Scope:** Global (user-level, all projects)
**Platforms:** Hermes Agent, Claude Code, OpenAI Codex CLI

---

## Overview

Ten skills that form a complete development workflow: architect a system, plan
a ticket, implement it, review the code, release to production, respond to
incidents, and audit health, security, performance, and tests. Each skill is
installed globally on all three platforms with platform-specific tool references.

```
architecture-spec → plan-to-linear → linear-ticket → code-review → release-deploy
     design             plan           implement       review         deploy
                          ↑               ↓                               ↓
                    test-strategy     repo-health                  incident-response
                    (designs tests)        ↓                          (postmortem)
                                      security-audit
                                      performance-audit
```

All skills are **global scope** — they live in user-level directories and apply to
every project, not per-repo.

---

## Skill Placement

| Platform | Skills directory | How skills load |
|---|---|---|
| **Hermes Agent** | `~/.hermes/skills/software-development/<name>/` | Auto-discovered at session start; cached per session |
| **Claude Code** | `~/.claude/skills/<name>/` | Auto-discovered from skills directory on launch |
| **Codex CLI** | `~/.codex/skills/<name>/` | Auto-discovered from skills directory on launch |

New skills appear on the **next session/launch** — the current session's skill
loader is cached and won't see mid-session additions.

---

## 1. architecture-spec

### What it does

Takes a raw requirement and produces a complete architecture specification
through an interactive question-and-answer process. Asks about the current
system (if one exists), gathers non-functional requirements and constraints,
presents multiple architecture options with trade-off matrices, and iterates
until the user confirms the spec is detailed enough to implement.

The output is a full architecture document with ADRs, component breakdown,
data flow, failure mode analysis, deployment topology, and a migration path —
ready to break into Linear projects and tickets via `plan-to-linear`.

### What it covers

- **Phase 1 — Requirements gathering:**
  - Functional requirements (core use cases, inputs/outputs, triggers, roles, boundaries)
  - Non-functional requirements with specific numbers (scale, latency, throughput, availability, security/compliance, consistency, read/write ratio, geographic distribution, data volume, durability)
  - Constraints inventory (team size/skills, budget, timeline, existing infrastructure, org policies, compliance constraints)
- **Phase 2 — Current state audit (if applicable):**
  - Architecture map (verified against codebase, not verbal description)
  - Pain points, what works, what's off-limits
  - Quick `codebase-onboarding` pass if codebase available
- **Phase 3 — Architecture options:**
  - 2-4 genuinely different options (not tweaks — different approaches)
  - Each with: description, component diagram, component list, tech stack, trade-off matrix scored against NFRs, cost estimate, risk assessment, spike candidate
  - User picks — skill does not recommend
- **Phase 4 — Deepen selected architecture (iterative):**
  - Component breakdown (responsibility, inputs, outputs, dependencies, tech, scaling, state)
  - Interface contracts (API endpoints, event schemas, data ownership)
  - Data flow (write path, read path, cache strategy, async paths, data stores)
  - Failure mode analysis (SPOFs, cascading failures, data loss, silent failures)
  - Deployment topology (environments, scaling, regions, network, CI/CD, observability)
  - Migration path (strangler fig, big bang, feature flag, parallel run, data migration, rollback)
  - Iterative deepening protocol — user says where to go deeper
- **Phase 5 — Architecture Decision Records (ADRs):**
  - Each major decision documented with context, options considered, decision, rationale, consequences
- **Phase 6 — Finalize spec:**
  - Complete spec written to file (`ARCHITECTURE.md`)
  - Open questions / deferred decisions listed explicitly
- **Phase 7 — Create Linear projects and tickets (optional):**
  - Architecture → Linear project
  - Components → Linear issues with acceptance criteria
  - Spike candidates → spike tickets
  - Phase ordering (foundational first, dependencies before dependents)

### How to run it

| Platform | Invocation |
|---|---|
| **Hermes** | Ask in chat: *"Design the architecture for [requirement]"* or *"Architect a system that does X"* |
| **Claude Code** | Ask in chat: *"Design the architecture for [requirement]"* or *"How should we build X?"* |
| **Codex** | Ask in chat: *"Architect a system for [requirement]"* or *"Design the architecture for X"* |

**Template:** `templates/spec.md` — the full spec document structure (requirements,
current state, options, selected architecture, components, interfaces, data flow,
failure modes, deployment, migration, ADRs, open questions, implementation plan).

**Prerequisites:** The requirement (even if vague). If existing system: codebase
access. Linear MCP for ticket creation (Phase 7).

**Note:** This is an **interactive skill** — it stops and waits for user input at
multiple gates (after requirements, after options, during deepening). The
questions are the skill; do not skip them.

---

## 2. plan-to-linear

### What it does

Plans a bug fix or feature end-to-end and creates a Linear ticket with the full
plan document as its description. Runs a mandatory **Minimal Change Challenge**
gate that forces the least amount of code possible — reuse existing utilities,
delete before adding, quantify the change footprint, reject speculative
abstractions.

### What it covers

- **Classification** — bug vs feature vs chore, with context gathering (repro
  steps for bugs, user stories for features)
- **Scope** — explicit in-scope and out-of-scope boundaries
- **Affected components** — verified by searching the codebase, never guessed
- **Risk & impact** — blast radius, backward compatibility, migration concerns
- **Acceptance criteria** — BDD Given/When/Then format, testable and binary
- **Test strategy** — unit, integration, e2e layers with edge cases from risk
  assessment
- **Implementation plan** — TDD Red → Green → Refactor, each step traceable to
  an acceptance criterion
- **Minimal Change Challenge (mandatory gate):**
  - Reuse-first audit — search for existing capabilities before writing new
  - Delete-before-add — identify dead code this change makes removable
  - Change footprint budget — files touched, lines added/removed
  - Alternatives considered — at least one heavier approach rejected with reason
  - Scope tightening — cut any step not traceable to an acceptance criterion
  - One-function rule — extend existing over creating new; justify new abstractions
- **Linear ticket creation** — resolves team, sets priority, creates issue with
  the plan as description, optional sub-issues for sub-tasks

### How to run it

| Platform | Invocation |
|---|---|
| **Hermes** | Ask in chat: *"Plan this feature and create a Linear ticket"* or *"Plan ISM-45"* — the skill triggers on planning requests |
| **Claude Code** | Ask in chat: *"Plan this bug fix and create a Linear ticket"* — Claude matches the skill from its description |
| **Codex** | Ask in chat: *"Plan this feature and create a Linear ticket"* — Codex matches the skill from its directory |

**Template:** `templates/plan.md` — the plan document structure that becomes the
Linear ticket description.

**Prerequisites:** Linear MCP server configured on each platform.

---

## 3. linear-ticket

### What it does

Takes a Linear ticket from identifier to shipped PR. Reads the full ticket
(including all comments), grounds it in the codebase, writes a plan, gets
approval, implements in a git worktree on the ticket's branch with TDD,
verifies against CI's actual commands, pushes, opens a PR, and moves the
ticket through Linear status (In Progress → In Review).

Includes an **overnight mode** (`--overnight`) for unattended multi-ticket
runs: one agent per ticket, parallel where independent, cascading PRs where
they depend on each other, one approval up front, morning report.

### What it covers

- **Phase 1** — Resolve ticket identifier (ISM-12, bare number, or URL), enter
  plan mode
- **Phase 2** — Read the whole ticket: description, all comments (including
  inline), sub-issues, parent, linked documents, attachments, blocked-by
  relations
- **Phase 3** — Ground in codebase: read repo rules (CLAUDE.md/AGENTS.md),
  find existing patterns, locate test infrastructure, find toolchain paths.
  Includes minimal-change discipline (reuse-first, delete-before-add, change
  footprint, alternatives considered). Write plan, get approval.
- **Phase 4** — Move ticket to In Progress, create git worktree on Linear's
  `gitBranchName`, install dependencies in worktree
- **Phase 5** — Implement test-first: write failing test, see it fail, minimal
  code to pass, refactor. Cover acceptance criteria, edge cases, failure paths
- **Phase 6** — Verify for real: run every CI step (not just tests), compare
  test totals against baseline, grep for collection/parse errors, run
  typecheck/lint. Nothing is done until this passes clean
- **Phase 7** — Commit (repo convention), confirm with user, push, open PR,
  move ticket to In Review, comment with PR link
- **Overnight mode (`--overnight`):**
  - Triage all tickets while user is awake, present run sheet, one approval
  - Dependency graph: blocks relations, parent/sub-issue order, file overlap,
    user-stated order
  - Dispatch: one agent per ticket, isolated worktrees/scratchpads/state dirs,
    parallel where independent, cascading PRs where dependent
  - Orchestrate: never end turn with agents running, handle "finished"
    notifications that are pauses, budget/parking per ticket
  - Morning report: per-ticket status, PR URLs, verification output, merge
    order, parked tickets, owed checks
  - `--merge` opt-in: land green PR stacks top-down, diagnose billing-refused
    CI vs real failures, assert gated tree = landed tree

### How to run it

| Platform | Invocation |
|---|---|
| **Hermes** | Ask in chat: *"Implement ISM-12"* or *"Take this ticket"* or paste a `linear.app/.../issue/...` URL |
| **Claude Code** | Ask in chat: *"Implement ISM-12"* or *"work on ticket ISM-12"* — Claude enters plan mode, reads the ticket, plans, and asks for approval before coding |
| **Codex** | Ask in chat: *"Implement ISM-12"* — Codex reads the ticket, writes a plan to a file, and waits for approval |

**Overnight mode:** Add `--overnight` to the request or say *"this is an
overnight run"* — the skill reads `OVERNIGHT.md` and switches to multi-ticket
orchestration.

**Supporting file:** `OVERNIGHT.md` — the full overnight orchestration protocol
(330 lines: triage, dependency graph, dispatch, orchestration, morning report,
merge mode, CI billing diagnosis).

**Prerequisites:** Linear MCP server, `gh` CLI, git repo. Codex requires `rmcp`
feature enabled for MCP.

---

## 4. code-review

### What it does

Structured code review that ties implementation back to the Linear plan,
enforces TDD discipline and minimal-change, runs 11 review lanes, and posts the
review as a Linear comment and/or PR comment. Two modes: pre-commit (your
changes before pushing) and PR review (a PR before merging).

Dispatches an **independent reviewer** with fresh context — never reviews its
own work. Fail-closed: unparseable response = fail.

### What it covers

**Two modes:**
- **Pre-commit** — reviews staged/unstaged changes (`git diff --cached`)
- **PR review** — reviews a PR via `gh pr diff <number>`

**11 review lanes (all run, even after CRITICAL found):**

| # | Lane | What it checks |
|---|---|---|
| 1 | Plan Compliance | Each acceptance criterion from the Linear ticket — SATISFIED / NOT SATISFIED / NOT FOUND |
| 2 | Minimal Change Verification | Planned change footprint vs actual, reuse audit, scope creep, bloat signal |
| 3 | TDD Discipline | Tests present, cover acceptance criteria, edge cases, failure paths, seen failing |
| 4 | Security | Secrets, injection, eval/exec, deserialization, path traversal, XSS, missing auth |
| 5 | Correctness | Logic errors, race conditions, null handling, resource leaks, async issues |
| 6 | Error Handling | Silent failures, swallowed errors, dangerous fallbacks (delegates to silent-failure-hunter patterns) |
| 7 | Framework-Specific | Conditional: React hooks/RSC, TS types, Python, Go, Rust — only if diff touches those files |
| 8 | Dependencies | New deps necessary? version pinning? supply chain flags? |
| 9 | Breaking Changes | Removed/renamed exports, signature changes, schema changes without migration |
| 10 | Documentation | Docstrings updated, README current, comments accurate, changelog entries |
| 11 | Git Hygiene | Commit convention, branch naming, debug code, commented-out code, secrets in diff |

**Severity gates:**
- CRITICAL → blocked (do not commit/merge)
- HIGH → changes requested (fix before commit)
- MEDIUM → warning (fix recommended, not blocking)
- LOW → suggestion (optional)

**Output:** Structured review document posted to Linear (`mcp__linear__create_comment`)
and/or GitHub PR (`gh pr comment`).

### How to run it

| Platform | Invocation |
|---|---|
| **Hermes** | Ask in chat: *"Review my changes before I push"* or *"Code review this PR"* or *"Review ISM-12 before merge"* |
| **Claude Code** | Ask in chat: *"Review my changes"* or *"Review PR #42"* — Claude dispatches an independent reviewer subagent via the `Task` tool |
| **Codex** | Ask in chat: *"Review my changes"* or *"Review PR #42"* — Codex dispatches an independent reviewer via `codex exec` |

**Template:** `templates/review.md` — the review document structure with all 11
lanes, verdict, and summary.

**Prerequisites:** Git repo with changes (pre-commit) or PR number/URL (PR
review), Linear MCP for ticket-linked reviews, `gh` CLI for PR comments.

---

## 5. repo-health

### What it does

Whole-repository maintainability review. Uses deterministic tooling to find
where the issues are, reads only the hotspots, runs 6 lenses, produces a health
scorecard with trend tracking, and creates actionable Linear tickets for
findings above the severity threshold.

**Not the same as `repo-audit`** — that finds bugs and security vulnerabilities.
This finds what makes the code harder to maintain and where to invest
refactoring effort.

### What it covers

**6 lenses:**

| # | Lens | What it finds |
|---|---|---|
| 1 | Dead Code & Bloat | Unused exports, orphan files, dead deps, stale TODOs, commented-out code, dead config/flags, over-abstraction |
| 2 | Refactoring Opportunities | Duplicated logic, long functions, God objects, deep nesting, feature envy, shotgun surgery, data clumps, primitive obsession, dead params, long param lists — each with the concrete refactor pattern |
| 3 | Decoupling Analysis | Coupling map, circular deps (with the edge to break and pattern to use), God modules, shared mutable state, dependency direction violations, suggested seams |
| 4 | Code Consistency Audit | Mixed error handling, mixed naming, mixed patterns, mixed framework usage, mixed export styles, config drift — each with which files use which approach |
| 5 | Enhancement Opportunities | Missing abstractions, missing observability, missing error boundaries, missing tests on high-risk modules, performance anti-patterns, missing validation |
| 6 | Technical Debt Inventory | TODO/FIXME/HACK catalog with age (git blame), category (bug/design/urgency), critical path flag, churn flag, trend vs baseline |

**Health scorecard** (with trend on reruns):
- Dead code count
- Duplication count
- Max inbound coupling (and which file)
- Circular dependencies
- Consistency violations
- Tech debt items (total / critical / high)
- Average debt age
- Test coverage on high-risk modules

**Baseline mode:** Reruns detect new vs still-open vs fixed debt, leading with
"New" findings. Rejected findings are excluded on future runs.

**Linear ticket creation:** Findings batched by area into `refactor: [area]`
tickets with sub-issues for large batches. Only CRITICAL/HIGH/MEDIUM get tickets;
LOW findings stay in the report.

**7 phases:**
1. Recon — deterministic tooling (`knip`, `madge`, `jscpd`, `vulture`, `pygount`, grep patterns)
2. Pick reading budget — top hotspots by coupling × complexity × debt
3. Six lenses — run all, verify each finding by reading surrounding code
4. Verify — re-read actual lines, confirm issue is real, write concrete recommendation
5. Health scorecard — compile metrics, compute trend if baseline exists
6. Report — write `.repo-health/REPORT.md`
7. Create Linear tickets (optional, if user approves) — batch findings, create issues

### How to run it

| Platform | Invocation |
|---|---|
| **Hermes** | Ask in chat: *"Review this repo for maintainability"* or *"Health check this codebase"* or *"Find refactoring opportunities"* |
| **Claude Code** | Ask in chat: *"Review this repo for tech debt"* or *"Audit code health"* — Claude runs the recon tools, reads hotspots, and produces the report |
| **Codex** | Ask in chat: *"Health check this repo"* or *"Find dead code and refactoring opportunities"* |

**Template:** `templates/report.md` — the report structure with health scorecard,
findings tables per lens, and recommended Linear tickets.

**Output:** `.repo-health/REPORT.md` in the repo root, plus `.repo-health/`
directory with recon data. On reruns, baseline comparison is automatic.

**Prerequisites:** Git repo. Optional but recommended: `knip`/`ts-prune` (dead
code), `madge` (circular deps), `jscpd` (duplication), `pygount` (metrics).
Linear MCP for ticket creation.

---

## Platform-Specific Notes

### Hermes Agent
- Skills are in `~/.hermes/skills/software-development/` with full frontmatter
  (version, author, license, platforms, metadata.hermes.tags, related_skills)
- Independent review uses `delegate_task`
- Shell commands via `terminal` tool; file operations via `read_file`,
  `write_file`, `patch`, `search_files`
- Linear integration via `mcp__linear__*` MCP tools

### Claude Code
- Skills are in `~/.claude/skills/<name>/`
- Minimal frontmatter (name + description only)
- Independent review uses the `Task` tool for subagent dispatch
- File operations via `Read`, `Write`, `Edit`, `Grep`
- Plan mode via `EnterPlanMode`/`ExitPlanMode`; worktree via `EnterWorktree`
- Linear integration via `mcp__linear__*` MCP tools

### Codex CLI
- Skills are in `~/.codex/skills/<name>/`
- Minimal frontmatter (name + description only)
- Independent review via separate `codex exec` process
- File operations via `grep`/`rg` and standard file I/O
- Plan approval is file-based (write plan, present, wait)
- Worktree via `cd` into `.codex/worktrees/`
- Linear MCP requires `rmcp` feature enabled: `codex mcp add linear --url https://mcp.linear.app/mcp`

---

## Linear MCP Setup

All three platforms use the same Linear MCP server:

| Platform | Setup command |
|---|---|
| **Hermes** | `hermes mcp add linear --url https://mcp.linear.app/mcp` (or configure in `~/.hermes/config.yaml`) |
| **Claude Code** | `claude mcp add linear --url https://mcp.linear.app/mcp` |
| **Codex** | `codex mcp add linear --url https://mcp.linear.app/mcp` (requires `rmcp` feature) |

The MCP server provides: `get_issue`, `get_comments`, `get_teams`,
`get_workflow_states`, `create_issue`, `update_issue`, `create_comment`,
`search_issues`, and `linear_ops_run` for advanced operations.

---

## File Inventory

| Skill | Files per platform |
|---|---|
| `architecture-spec` | `SKILL.md` + `templates/spec.md` |
| `plan-to-linear` | `SKILL.md` + `templates/plan.md` |
| `linear-ticket` | `SKILL.md` + `OVERNIGHT.md` |
| `code-review` | `SKILL.md` + `templates/review.md` |
| `release-deploy` | `SKILL.md` + `templates/release.md` |
| `repo-health` | `SKILL.md` + `templates/report.md` |
| `security-audit` | `SKILL.md` + `templates/report.md` |
| `performance-audit` | `SKILL.md` + `templates/report.md` |
| `incident-response` | `SKILL.md` + `templates/postmortem.md` |
| `test-strategy` | `SKILL.md` + `templates/report.md` |

Total: 10 skills × 3 platforms = 30 installations, 60 files.
