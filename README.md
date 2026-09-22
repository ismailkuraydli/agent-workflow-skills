# Agent Workflow Skills

A collection of AI agent skills that form a complete development workflow: **architect a system → plan a ticket → implement it → review the code → release to production → respond to incidents → audit health, security, performance, and tests → generate docs → upgrade dependencies**. Each skill is installed globally on three platforms with platform-specific tool references.

```
architecture-spec → plan-to-linear → linear-ticket → code-review → release-deploy
     design             plan           implement       review         deploy
                          ↑               ↓                               ↓
                    test-strategy     repo-health                  incident-response
                    (designs tests)        ↓                          (postmortem)
                                      security-audit
                                      performance-audit
```

## Skills

| # | Skill | What it does |
|---|---|---|
| 1 | **architecture-spec** | Takes a requirement, asks questions, presents multiple architecture options with trade-off matrices, iterates until fully spec'd. Outputs ADRs, component breakdown, data flow, failure modes, deployment topology. |
| 2 | **plan-to-linear** | Plans a bug fix or feature with TDD, BDD acceptance criteria, and a mandatory minimal-change challenge gate. Creates a Linear ticket with the full plan document. |
| 3 | **linear-ticket** | Takes a Linear ticket from identifier to shipped PR. Reads the full ticket, grounds in codebase, plans, implements in a git worktree with TDD, verifies against CI, pushes, opens PR. Includes overnight multi-ticket mode. |
| 4 | **code-review** | Structured code review with 11 lanes (plan compliance, minimal-change, TDD, security, correctness, error handling, framework-specific, dependencies, breaking changes, docs, git hygiene). Posts review to Linear and/or GitHub PR. |
| 5 | **release-deploy** | Ship releases: version bump (semver), changelog from merged PRs, pre-deploy checklist, deploy with strategy selection (rolling/blue-green/canary), smoke tests, rollback plan, Linear ticket closure. |
| 6 | **repo-health** | Whole-repo maintainability review: dead code, refactoring opportunities, decoupling analysis, code consistency, enhancement opportunities, and tech debt inventory. Produces health scorecard with trends and creates Linear tickets. |
| 7 | **security-audit** | Contextual security audit: maps attack surface, traces features to PRs, fetches live CVEs for the exact dependency stack, retrieves technology-specific best practices, OWASP/CWE/CVSS mapping, compliance mapping (GDPR/HIPAA/SOC2/PCI). |
| 8 | **performance-audit** | Finds performance bottlenecks: N+1 queries, memory leaks, slow hot paths, oversized bundles, unnecessary re-renders, sync I/O in async paths, missing pagination. Measures against latency budgets. |
| 9 | **incident-response** | Triage incidents (SEV-1 to SEV-4), stabilize (rollback or fix-forward), root cause analysis (git bisect, deploy diff), blameless postmortem with timeline and action items, Linear tickets for fixes, runbook update. |
| 10 | **test-strategy** | Maps test coverage against risk, finds tests that can't fail, detects excessive mocking and snapshot tests, identifies missing test types (failure modes, edge cases), designs the test pyramid per feature, creates a test plan with acceptance criteria. |
| 11 | **docs-gen** | Generates and maintains documentation from code: API docs (OpenAPI/Swagger), READMEs, CHANGELOGs, runbooks, and inline docstrings. Detects documentation drift (docs that don't match code) and flags mismatches. |
| 12 | **dep-upgrade** | Scans for outdated dependencies, classifies by risk (patch/minor/major/security), reads changelogs for breaking changes, upgrades one batch at a time with test verification against a baseline, creates a PR per batch. |

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
    │   ├── architecture-spec/             # SKILL.md + templates/spec.md
    │   ├── plan-to-linear/                # SKILL.md + templates/plan.md
    │   ├── linear-ticket/                 # SKILL.md + OVERNIGHT.md
    │   ├── code-review/                   # SKILL.md + templates/review.md
    │   ├── release-deploy/                # SKILL.md + templates/release.md
    │   ├── repo-health/                   # SKILL.md + templates/report.md
    │   ├── security-audit/                # SKILL.md + templates/report.md
    │   ├── performance-audit/             # SKILL.md + templates/report.md
    │   ├── incident-response/             # SKILL.md + templates/postmortem.md
    │   ├── test-strategy/                 # SKILL.md + templates/report.md
    │   ├── docs-gen/                      # SKILL.md + templates/report.md
    │   └── dep-upgrade/                   # SKILL.md + templates/report.md
    ├── claude/                            # Claude Code versions (same structure)
    └── codex/                             # Codex CLI versions (same structure)
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

- **Git** — all skills assume a git repository. [Install](https://git-scm.com/downloads)
- **GitHub CLI (`gh`)** — used by `linear-ticket` (push branches, open PRs, create worktrees), `code-review` (fetch PR diffs, post review comments), `release-deploy` (PR history, GitHub releases), and `security-audit` (GitHub Security Advisories). [Install](https://cli.github.com/)
  ```bash
  # Install (macOS)
  brew install gh

  # Install (Linux)
  sudo apt install gh  # Debian/Ubuntu
  sudo dnf install gh  # Fedora

  # Authenticate
  gh auth login

  # Verify
  gh auth status
  ```
- **Linear MCP server** — used by all skills for ticket creation, status updates, and comment posting. Configure on each platform:
  - Hermes: `hermes mcp add linear --url https://mcp.linear.app/mcp`
  - Claude Code: `claude mcp add linear --url https://mcp.linear.app/mcp`
  - Codex: `codex mcp add linear --url https://mcp.linear.app/mcp` (requires `rmcp` feature)

## Optional Integrations

### Graphify — Code Knowledge Graph

[Graphify](https://graphify.com) maps your codebase into an on-device knowledge graph your agent queries instead of grepping. It parses 36 languages with tree-sitter (no API calls for code), tags every edge as EXTRACTED/INFERRED/AMBIGUOUS, and outputs an interactive graph + architecture report.

**Why use it with these skills:**
- `security-audit` uses Graphify MCP tools (`get_pr_impact`, `triage_prs`) to trace which PRs touched security-critical code
- `code-review` can query the graph to understand how a diff connects to the rest of the system
- `repo-health` uses the coupling map to find God modules and circular dependencies
- Any skill can use `query_graph` to answer "what connects X to Y?" without grepping

**Install:**
```bash
# Install the CLI (requires uv or pipx)
uv tool install "graphifyy[mcp]"

# Register with your assistants
graphify install                    # Claude Code (default)
graphify install --platform codex   # Codex

# For Hermes, add the MCP server to config:
hermes config set mcp_servers.graphify.command "/path/to/graphify/python"
hermes config set mcp_servers.graphify.args '["-m", "graphify.serve", "graphify-out/graph.json"]'
```

**Build the graph** (run inside your repo):
```bash
# Code-only (no API key needed, AST extraction only)
graphify . --code-only

# With docs/papers/images (needs an LLM API key for semantic extraction)
export OPENAI_API_KEY=***          # or ANTHROPIC_API_KEY, DEEPSEEK_API_KEY
export OPENAI_BASE_URL=https://openrouter.ai/api/v1   # OpenRouter works too
export OPENAI_MODEL=anthropic/claude-sonnet-4
graphify .

# Label communities with real names (uses the LLM backend)
graphify label .
```

**Auto-update on commit:**
```bash
graphify hook install    # adds post-commit and post-checkout git hooks
```

**Query from the CLI:**
```bash
graphify query "what connects auth to the database?"
graphify path "UserService" "DatabasePool"
graphify explain "RateLimiter"
graphify god-nodes           # most connected modules
graphify prs                 # map open PRs onto the graph
```

**Output files** (in `graphify-out/`):
| File | What it is |
|---|---|
| `graph.json` | The queryable knowledge graph |
| `graph.html` | Interactive visual graph (open in browser) |
| `GRAPH_REPORT.md` | Architecture report (god nodes, cross-file connections) |

Add `graphify-out/` to `.gitignore` — it's generated, not source.

### Caveman — Token Compression

[Caveman](https://github.com/JuliusBrussee/caveman) cuts ~65-75% of output tokens by making your agent respond in compressed "caveman-speak" while keeping full technical accuracy. It also includes a proxy that compresses input context before the agent reads it.

**Two modes:**
1. **Skill (output compression)** — agent replies in terse prose. Toggle with `/caveman` (on) and `/caveman off` (off). Levels: `lite`, `full` (default), `ultra`.
2. **Proxy (input compression)** — MCP middleware that compresses context (files, diffs, MCP results) before the agent reads it, with byte-exact recovery. Runs automatically in the background.

**Install:**
```bash
# Install the CLI
npm install -g @caveman-ai/cli

# Install binaries (proxy, engine, mcp, etc.)
caveman setup --install

# Enable the proxy for Claude Code
caveman setup --agent-native claude

# Enable the proxy for Codex (requires Codex CLI installed)
caveman setup --agent-native codex

# Install the skill on all platforms
npx skills add JuliusBrussee/caveman
```

**For Hermes proxy** (if Hermes binary is not on PATH):
```bash
export PATH="$HOME/.local/bin:$PATH"
caveman hermes
```

**Usage:**
```bash
# Check status and savings
caveman status       # what the proxy is doing
caveman stats        # session + lifetime token savings

# Toggle the skill in chat (Claude Code or Hermes)
/caveman             # enable terse output
/caveman lite        # less aggressive
/caveman ultra       # most aggressive
/caveman off         # disable — back to normal output

# Remove the proxy
caveman setup --agent-native claude --remove
```

**When to use / not use:**
- ✅ Casual coding sessions, quick questions, debugging — saves tokens on filler
- ✅ Long sessions where context window fills up — proxy compresses input
- ⚠️ Turn OFF when running skills that produce structured documents (`plan-to-linear`, `architecture-spec`, `code-review`) — those need full prose. Use `/caveman off` before running them.

## Key Design Principles

1. **Minimal change by default** — `plan-to-linear` and `code-review` both enforce a minimal-change challenge: reuse existing code, delete before adding, quantify the footprint, reject speculative abstractions.

2. **Interactive, not autonomous** — `architecture-spec` stops at multiple gates to ask questions. The questions shape the output. It never generates the whole spec in one pass.

3. **Independent review** — `code-review` dispatches a fresh subagent with no shared context. No agent reviews its own work.

4. **Evidence or it doesn't ship** — `repo-health`, `security-audit`, `performance-audit`, and `code-review` require `file:line` and concrete recommendations. No "consider refactoring this."

5. **Workflow integration** — skills feed into each other: `architecture-spec` breaks into Linear tickets → `plan-to-linear` plans each ticket → `linear-ticket` implements → `code-review` reviews → `release-deploy` ships → `incident-response` handles breakage → `repo-health` / `security-audit` / `performance-audit` / `test-strategy` audit the result.

6. **Blameless postmortems** — `incident-response` focuses on the system, not the people. "The deploy pipeline didn't run the integration test suite" is a root cause. "Bob deployed bad code" is not.

7. **Risk-weighted testing** — `test-strategy` maps tests to risk, not to coverage percentage. 100% coverage with tests that can't fail is worse than 60% coverage with tests that actually verify behavior.

## License

MIT
