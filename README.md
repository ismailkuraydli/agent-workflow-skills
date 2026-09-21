# Agent Workflow Skills

A collection of AI agent skills that form a complete development workflow: **architect a system → plan a ticket → implement it → review the code → audit the repo's health**. Each skill is installed globally on three platforms with platform-specific tool references.

```
architecture-spec → plan-to-linear → linear-ticket → code-review → repo-health
     design             plan           implement       review         audit
```

## Skills

| # | Skill | What it does |
|---|---|---|
| 1 | **architecture-spec** | Takes a requirement, asks questions, presents multiple architecture options with trade-off matrices, iterates until fully spec'd. Outputs ADRs, component breakdown, data flow, failure modes, deployment topology. |
| 2 | **plan-to-linear** | Plans a bug fix or feature with TDD, BDD acceptance criteria, and a mandatory minimal-change challenge gate. Creates a Linear ticket with the full plan document. |
| 3 | **linear-ticket** | Takes a Linear ticket from identifier to shipped PR. Reads the full ticket, grounds in codebase, plans, implements in a git worktree with TDD, verifies against CI, pushes, opens PR. Includes overnight multi-ticket mode. |
| 4 | **code-review** | Structured code review with 11 lanes (plan compliance, minimal-change, TDD, security, correctness, error handling, framework-specific, dependencies, breaking changes, docs, git hygiene). Posts review to Linear and/or GitHub PR. |
| 5 | **repo-health** | Whole-repo maintainability review: dead code, refactoring opportunities, decoupling analysis, code consistency, enhancement opportunities, and tech debt inventory. Produces health scorecard with trends and creates Linear tickets. |

## Platforms

Each skill is adapted for three AI agent platforms:

| Platform | Skills directory | Tool references |
|---|---|---|
| **Hermes Agent** | `~/.hermes/skills/software-development/<name>/` | `search_files`, `read_file`, `write_file`, `patch`, `delegate_task` |
| **Claude Code** | `~/.claude/skills/<name>/` | `Grep`, `Read`, `Write`, `Edit`, `Task` |
| **Codex CLI** | `~/.codex/skills/<name>/` | `grep`/`rg`, file I/O, `codex exec` |

All three use the same **Linear MCP server** (`https://mcp.linear.app/mcp`) for ticket creation and status updates.

## Repository Structure

```
agent-workflow-skills/
├── README.md
├── LICENSE
├── docs/
│   └── agent-workflow-skills.md          # Full documentation with run instructions
└── skills/
    ├── hermes/                            # Hermes Agent versions
    │   ├── architecture-spec/
    │   │   ├── SKILL.md
    │   │   └── templates/spec.md
    │   ├── plan-to-linear/
    │   │   ├── SKILL.md
    │   │   └── templates/plan.md
    │   ├── linear-ticket/
    │   │   ├── SKILL.md
    │   │   └── OVERNIGHT.md
    │   ├── code-review/
    │   │   ├── SKILL.md
    │   │   └── templates/review.md
    │   └── repo-health/
    │       ├── SKILL.md
    │       └── templates/report.md
    ├── claude/                            # Claude Code versions
    │   ├── architecture-spec/
    │   ├── plan-to-linear/
    │   ├── linear-ticket/
    │   ├── code-review/
    │   └── repo-health/
    └── codex/                             # Codex CLI versions
        ├── architecture-spec/
        ├── plan-to-linear/
        ├── linear-ticket/
        ├── code-review/
        └── repo-health/
```

## Installation

### Install all skills to all platforms

```bash
# Clone
git clone https://github.com/ismailkuraydli/agent-workflow-skills.git
cd agent-workflow-skills

# Install to Hermes
cp -r skills/hermes/* ~/.hermes/skills/software-development/

# Install to Claude Code
cp -r skills/claude/* ~/.claude/skills/

# Install to Codex
cp -r skills/codex/* ~/.codex/skills/
```

### Install a single skill to a single platform

```bash
# Example: install only plan-to-linear to Claude Code
cp -r skills/claude/plan-to-linear ~/.claude/skills/
```

### Prerequisites

- **Linear MCP server** configured on each platform:
  - Hermes: `hermes mcp add linear --url https://mcp.linear.app/mcp`
  - Claude Code: `claude mcp add linear --url https://mcp.linear.app/mcp`
  - Codex: `codex mcp add linear --url https://mcp.linear.app/mcp` (requires `rmcp` feature)
- **`gh` CLI** for GitHub PR operations (used by `linear-ticket` and `code-review`)
- **Git** — all skills assume a git repository

## Key Design Principles

1. **Minimal change by default** — `plan-to-linear` and `code-review` both enforce a minimal-change challenge: reuse existing code, delete before adding, quantify the footprint, reject speculative abstractions.

2. **Interactive, not autonomous** — `architecture-spec` stops at multiple gates to ask questions. The questions shape the output. It never generates the whole spec in one pass.

3. **Independent review** — `code-review` dispatches a fresh subagent with no shared context. No agent reviews its own work.

4. **Evidence or it doesn't ship** — `repo-health` and `code-review` require `file:line` and concrete recommendations. No "consider refactoring this."

5. **Workflow integration** — skills feed into each other: `architecture-spec` breaks into Linear tickets → `plan-to-linear` plans each ticket → `linear-ticket` implements → `code-review` reviews → `repo-health` audits the result.

## License

MIT
