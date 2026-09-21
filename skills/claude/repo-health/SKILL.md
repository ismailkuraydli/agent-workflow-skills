---
name: repo-health
description: "Full repo review for dead code, refactoring, decoupling, consistency."
---
# Repo Health

Whole-repository maintainability review. Finds dead code, refactoring opportunities, coupling problems, consistency violations, enhancement opportunities, and technical debt — then produces a health scorecard and actionable Linear ticket suggestions.

**Not the same as `repo-audit`**, which finds bugs and security vulnerabilities. This skill finds what makes the code harder to maintain and where to invest refactoring effort.

**Core constraint:** A repo does not fit in context. Use deterministic tooling to find where the issues are, then spend the reading budget only there. Every finding needs `file:line` and a concrete recommendation — no "consider refactoring this."

## When to Use

- User asks to review the whole repo for maintainability, tech debt, or code health
- User asks to find refactoring opportunities, dead code, or decoupling issues
- User asks for a code consistency audit
- User asks to catalog technical debt
- Periodic health check (monthly/quarterly)

## Don't Use For

- Bug finding or security auditing → use `repo-audit`
- Single diff/PR review → use `code-review`
- Onboarding/understanding a codebase → use `codebase-onboarding`
- LOC counting → use `codebase-inspection`

## Prerequisites

- A git repository
- Linear MCP tools for ticket creation (`mcp__linear__get_teams`, `mcp__linear__create_issue`)
- Optional but recommended: `knip`/`ts-prune` (dead code), `madge` (circular deps), `jscpd` (duplication), `dependency-cruiser` (coupling)

## Procedure

### Phase 1 — Recon (deterministic, no LLM reading)

Run deterministic analysis tools to map the codebase. All output goes to `.repo-health/`.

**1.1 — Codebase metrics:**
```bash
# Language breakdown, file count, LOC
pygount --format=summary --folders-to-skip=".git,node_modules,venv,.venv,__pycache__,dist,build,.next" .
```

**1.2 — Dead code detection (use whatever is available):**
```bash
# JS/TS
npx knip --reporter json 2>/dev/null || npx ts-prune 2>/dev/null
# Python
python -m vulture --min-confidence 80 . 2>/dev/null
# Go
go vet ./... 2>&1 | grep -i "declared but not used\|unused"
# Rust
cargo +nightly udeps 2>/dev/null
```

If no tool is available, fall back to grep patterns:
```bash
# Unused exports (JS/TS) — exported but never imported
grep -rn "export " src/ | while read line; do
  symbol=$(echo "$line" | sed -n 's/.*export \(const\|function\|class\|default\) \([a-zA-Z_][a-zA-Z0-9_]*\).*/\2/p')
  [ -n "$symbol" ] && ! grep -rq "\b$symbol\b" src/ --exclude=$(echo "$line" | cut -d: -f1) && echo "UNUSED: $line"
done

# Unused Python functions — defined but never called
grep -rn "^def \|^\s\sdef " --include="*.py" . | while read line; do
  func=$(echo "$line" | sed -n 's/.*def \([a-zA-Z_][a-zA-Z0-9_]*\).*/\1/p')
  [ -n "$func" ] && [ $(grep -r "\b$func\b" --include="*.py" . | wc -l) -le 1 ] && echo "UNUSED: $line"
done
```

**1.3 — Duplication detection:**
```bash
npx jscpd --min-lines 6 --format "js,ts,py,go,rs" --reporters json 2>/dev/null
```

**1.4 — Circular dependency detection:**
```bash
# JS/TS
npx madge --circular --extensions js,ts,tsx,jsx src/ 2>/dev/null
# Python (import graph)
python -c "
import ast, os, sys
from collections import defaultdict
deps = defaultdict(set)
for root, _, files in os.walk('.'):
    for f in files:
        if not f.endswith('.py'): continue
        path = os.path.join(root, f)
        try:
            tree = ast.parse(open(path).read())
        except: continue
        mod = path.replace('/','..')[:-3]
        for node in ast.walk(tree):
            if isinstance(node, ast.Import):
                for a in node.names: deps[mod].add(a.name)
            elif isinstance(node, ast.ImportFrom) and node.module:
                deps[mod].add(node.module)
# Simple cycle detection
def find_cycles(graph):
    cycles = []
    def dfs(node, visited, path):
        if node in path:
            cycles.append(path[path.index(node):] + [node])
            return
        if node in visited: return
        visited.add(node)
        for dep in graph.get(node, set()):
            dfs(dep, visited, path + [node])
    for n in graph: dfs(n, set(), [])
    return cycles
for c in find_cycles(deps): print(' -> '.join(c))
" 2>/dev/null
```

**1.5 — Coupling map (inbound dependency count per module):**
```bash
# Count how many files import each module — high count = God module
grep -rn "import " --include="*.ts" --include="*.js" --include="*.tsx" --include="*.py" src/ | \
  sed -n 's/.*import.*from ["'\'']\([^"'\'']\+\).*/\1/p; s/.*import \([a-zA-Z_][a-zA-Z0-9_.]*\).*/\1/p' | \
  sort | uniq -c | sort -rn | head -30
```

**1.6 — Tech debt catalog:**
```bash
# All TODO/FIXME/HACK/WORKAROUND/TEMP/XXX with file:line
grep -rn "TODO\|FIXME\|HACK\|WORKAROUND\|TEMP:\|XXX" --include="*.ts" --include="*.js" --include="*.tsx" --include="*.py" --include="*.go" --include="*.rs" . | \
  grep -v node_modules | grep -v ".min." > .repo-health/tech-debt.tsv

# Age of each debt item (how old is the line?)
while IFS= read -r line; do
  file=$(echo "$line" | cut -d: -f1)
  lineno=$(echo "$line" | cut -d: -f2)
  age=$(git blame -L "$lineno,$lineno" --format="%ai" "$file" 2>/dev/null | head -1)
  echo "$age|$line"
done < .repo-health/tech-debt.tsv > .repo-health/tech-debt-aged.tsv
```

**1.7 — Consistency signals:**
```bash
# Mixed error handling (throw vs return null vs Result type)
echo "=== throw ===" && grep -rn "throw " --include="*.ts" --include="*.py" src/ | wc -l
echo "=== return null/None ===" && grep -rn "return null\|return None" --include="*.ts" --include="*.py" src/ | wc -l
echo "=== Result type ===" && grep -rn "Result<\|Either<\|Ok(\|Err(" --include="*.ts" --include="*.py" src/ | wc -l

# Mixed naming (camelCase vs snake_case in same layer)
echo "=== camelCase functions ===" && grep -rn "function [a-z][a-zA-Z]*(" --include="*.ts" src/ | wc -l
echo "=== snake_case functions ===" && grep -rn "function [a-z_]*(" --include="*.ts" src/ | wc -l

# Mixed HTTP clients
echo "=== fetch ===" && grep -rn "fetch(" --include="*.ts" --include="*.js" src/ | wc -l
echo "=== axios ===" && grep -rn "axios" --include="*.ts" --include="*.js" src/ | wc -l

# Mixed export style
echo "=== default exports ===" && grep -rn "export default" --include="*.ts" --include="*.tsx" src/ | wc -l
echo "=== named exports ===" && grep -rn "export {" --include="*.ts" --include="*.tsx" src/ | wc -l
```

**1.8 — Complexity hotspots:**
```bash
# Long functions (30+ lines)
grep -rn "" --include="*.ts" --include="*.py" --include="*.go" src/ | \
  awk -F: '{print $1}' | sort -u | while read f; do
    awk '/^[[:space:]]*(function|def|func) /{name=$0; start=NR} /^}/{if(start && NR-start>30) print FILENAME":"start" "name" ("NR-start" lines)"; start=0}' "$f"
  done

# Deep nesting (4+ levels of indentation)
grep -rn "" --include="*.ts" --include="*.py" --include="*.go" src/ | \
  awk -F: 'BEGIN{max=0} {n=gsub(/\t/,"",$3); if(n>=4) print FILENAME":"$1":"n" levels"}' | sort -t: -k3 -rn | head -20
```

Record all output under `.repo-health/`. Note which tools were unavailable — that becomes a coverage note in the report. Offer to add `.repo-health/` to `.gitignore`.

### Phase 2 — Pick the reading budget

From the recon data, choose files to read in full:

- **Top 15 hotspot files** — highest coupling, longest functions, most duplication
- **God modules** — top 5 by inbound dependency count
- **High-churn files with debt** — TODOs in files changed frequently (git log --since)
- **Entry points** — server bootstrap, route registration, DB layer, config

State the budget in the report: "Read 22 of 480 files, selected by coupling × complexity × debt density."

### Phase 3 — Six lenses (run in order)

Each lens produces findings. Use `Grep` and `Read` to verify — never report from grep alone.

#### Lens 1 — Dead Code & Bloat

- Unused exports, orphan files, unreachable branches
- Unused dependencies, dev deps in production code
- Stale TODOs (over 6 months old via git blame)
- Commented-out code blocks
- Feature flags always on/off, dead config keys, dead env vars
- Over-abstraction: interfaces with one implementation, wrapper classes adding nothing, config for config

Rank by removal effort: "delete + update imports" is cheap; "unravel dependency chain" is expensive.

#### Lens 2 — Refactoring Opportunities

Code smell detection, ranked by impact ÷ effort:

- **Duplicated logic** — same algorithm copy-pasted across files (structural, not just text)
- **Long functions** — over 50 lines doing more than one thing
- **Large classes/modules** — God objects with too many responsibilities
- **Deep nesting** — 4+ levels of conditionals/loops
- **Feature envy** — method using another class's data more than its own
- **Shotgun surgery** — one change requires touching many files
- **Data clumps** — same group of params passed around together (extract into a type)
- **Primitive obsession** — using strings/ints where a value object would be clearer
- **Dead parameters** — function params never used by callers
- **Long parameter lists** — 5+ params (use a config object)

For each: state the smell, the file:line, the concrete refactor (extract method, extract class, replace conditional with polymorphism, etc.), and the impact.

#### Lens 3 — Decoupling Analysis

- **Coupling map** — which modules depend on which, density per module
- **Circular dependencies** — from `madge` or the Python import graph
- **God modules** — modules imported by 10+ others (everything routes through here)
- **Shared mutable state** — global singletons, module-level state, shared caches
- **Dependency direction violations** — utils importing from domain, domain importing from UI
- **Transitive chains** — A doesn't import C but gets it through B
- **Suggested seams** — where to cut: extract interface, invert dependency, extract module, introduce event bus

For each circular dep: trace the cycle, state which edge to break, and what pattern to use (dependency inversion, extract shared module, event-driven).

#### Lens 4 — Code Consistency Audit

From the Phase 1.7 signals, verify each inconsistency by reading the actual files:

- **Mixed error handling** — some modules throw, others return null, others use Result types in the same layer
- **Mixed naming** — camelCase and snake_case in the same layer
- **Mixed patterns** — different approaches to the same problem (some routes use middleware, others inline auth)
- **Mixed framework usage** — some fetch via axios, some via fetch, some via a wrapper
- **Mixed export style** — default exports and named exports in the same package
- **Config drift** — same setting defined in multiple places with different values

For each: state what the two approaches are, which files use which, and which is the established pattern (the majority wins — the fix is to align the minority).

#### Lens 5 — Enhancement Opportunities

- **Missing abstractions** — repeated inline logic that should be a function
- **Missing observability** — I/O paths with no logging/metrics/tracing
- **Missing error boundaries** — async paths with no error handling
- **Missing tests** — high-risk modules (auth, payment, data mutation) with no test coverage
- **Performance anti-patterns** — N+1 queries, unbounded fetches, sync I/O in hot paths, unnecessary re-renders, missing pagination
- **Missing validation** — API endpoints accepting unvalidated input

For each: state what's missing, the file:line, and what to add.

#### Lens 6 — Technical Debt Inventory

From the Phase 1.6 catalog:

- **Categorize:** bug debt (FIXME/XXX), design debt (TODO), urgency debt (HACK/WORKAROUND)
- **Age:** how old is each item (git blame date)
- **Churn:** is the file containing the debt still being actively modified? (git log --oneline --since for that file)
- **Criticality:** is the debt in a critical path? (auth, payment, data integrity, security)
- **Trend:** is debt being paid down or accumulating? (compare counts to previous `.repo-health/` run if baseline exists)

Flag debt that is: old AND in high-churn files (getting worse), or in critical-path files (high risk).

### Phase 4 — Verify

For each candidate finding:
1. Re-read the actual lines — not the grep hit, the surrounding function
2. Confirm the issue is real and not already handled elsewhere
3. Write the concrete recommendation: what to do, not just what's wrong
4. Assign severity and effort estimate

Drop rate is normally 30-50% — false positives from grep patterns are common.

### Phase 5 — Health Scorecard

Compile metrics into a scorecard:

| Metric | Value | Trend |
|---|---|---|
| Dead code (unused exports/functions) | N | ↑/↓/→ from baseline |
| Duplication (duplicated blocks) | N | ↑/↓/→ |
| Max inbound coupling | N (file) | ↑/↓/→ |
| Circular dependencies | N | ↑/↓/→ |
| Consistency violations | N | ↑/↓/→ |
| Tech debt items (total / critical / high) | N | ↑/↓/→ |
| Avg debt age | N months | ↑/↓/→ |
| Test coverage (high-risk modules) | N% | ↑/↓/→ |

If a baseline `.repo-health/` exists from a previous run, compute the trend. Otherwise trend = "first run."

### Phase 6 — Report

Write `.repo-health/REPORT.md` using `templates/report.md`. Include:

1. **Health scorecard** at the top
2. **Findings by lens** — ranked by severity, each with file:line, impact, effort, recommendation
3. **Recommended Linear tickets** — batched by area (see below)
4. **Coverage note** — which tools ran, which were unavailable, files read vs total

In chat: the scorecard, top 5 findings, and the report path. Do not paste the full report.

### Phase 7 — Create Linear Tickets (optional, if user approves)

Batch related findings into Linear tickets:

1. Resolve the team: `mcp__linear__get_teams`
2. For each batch, create an issue:
   - **Title:** `refactor: [area] — [summary]` (e.g., "refactor: extract auth middleware — 4 routes inline auth checks")
   - **Description:** the findings, file:line for each, the recommended action, and the impact
   - **Priority:** based on severity (CRITICAL=1, HIGH=2, MEDIUM=3, LOW=4)
   - **Label:** "tech-debt" or "refactor" if available
3. For large batches, create a parent umbrella ticket with sub-issues for each finding
4. Confirm ticket URLs back to the user

**Batching rules:**
- Findings in the same file/module → one ticket
- Same refactor type across multiple files (e.g., "extract error handler" in 8 files) → one ticket with a checklist
- Circular dependency clusters → one ticket per cycle
- Dead code in one area → one "cleanup: [area]" ticket

## Severity Levels

| Level | Meaning | Action |
|---|---|---|
| CRITICAL | Debt in critical path (auth, payment, data integrity) or coupling that blocks all change | Create Linear ticket immediately |
| HIGH | Actively slowing development, or will cause bugs soon | Create Linear ticket, schedule next sprint |
| MEDIUM | Maintainability cost, not urgent | Create Linear ticket, backlog |
| LOW | Minor, nice-to-have | List in report, no ticket |

## Baseline Mode

If `.repo-health/REPORT.md` and `.repo-health/findings.json` already exist:
1. Load as baseline
2. Match new findings to old by file + category + summary similarity
3. Report as **New**, **Still open**, **Fixed** (in baseline, absent now — verify it's actually fixed)
4. Lead with **New** and **Trend** — that's what matters on a rerun
5. Findings the user rejected last time (`status: wontfix` in JSON) are excluded

## Pitfalls

- **Reading the whole repo** — never. Use recon to find hotspots, read only those.
- **Grep without verification** — grep finds patterns, not issues. Always read the surrounding code before reporting.
- **Reporting style preferences as findings** — formatting, import order, and style are the formatter's job. Only report consistency violations that affect different approaches to the same problem.
- **Suggesting rewrites nobody asked for** — "rewrite this in X framework" is not a finding. The recommendation must work within the existing stack.
- **Creating tickets for LOW findings** — flooding Linear with minor cleanup tickets wastes triage time. Only CRITICAL/HIGH/MEDIUM get tickets.
- **Missing the trend** — on a rerun, the scorecard trend is the most valuable output. If no baseline exists, say "first run" rather than omitting the column.
- **Dead code false positives** — dynamically imported modules, string-referenced exports, and plugin systems won't show up as "used" by static analysis. Verify before reporting as dead.
- **Circular dep false positives** — test files importing from production code are not real cycles. Filter test files from the import graph.

## Verification

- [ ] Recon phase completed — all available tools ran, unavailable ones noted
- [ ] Reading budget stated — "Read N of M files"
- [ ] All 6 lenses run — each has findings or "none found"
- [ ] Every finding has file:line, concrete recommendation, severity, and effort
- [ ] Health scorecard compiled with metrics
- [ ] Report written to `.repo-health/REPORT.md`
- [ ] Linear tickets created (if user approved) — URLs returned
- [ ] Coverage note documents what was and wasn't scanned
