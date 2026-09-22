---
name: performance-audit
description: "Find performance bottlenecks: N+1, memory leaks, slow paths, bundle."
version: 0.1.0
author: Ismail Kuraydli, Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [performance, profiling, bottleneck, n+1, memory-leak, bundle-size, latency]
    related_skills: [repo-health, security-audit, repo-audit, code-review]
---

# Performance Audit

Finds performance bottlenecks in a codebase: N+1 queries, memory leaks, slow
hot paths, oversized bundles, unnecessary re-renders, sync I/O in async paths,
and missing pagination. Profiles the codebase, identifies hot paths, measures
against latency budgets, and benchmarks before/after fixes.

**Not the same as `repo-health`** (maintainability) or `security-audit`
(vulnerabilities). This is about speed, memory, and resource efficiency.

## When to Use

- User asks to check or improve performance
- User says "the app is slow" or "this endpoint takes too long"
- User asks to profile the codebase
- After a performance regression
- Before a launch or load test
- Periodic performance review

## Don't Use For

- Code quality / maintainability → use `repo-health`
- Security → use `security-audit`
- Single PR review → use `code-review`

## Prerequisites

- A codebase to audit
- `web_search` for technology-specific performance best practices
- Optional: profiling tools (`pytest-benchmark`, `k6`, `wrk`, `lighthouse`, `clinic.js`)
- Optional: Graphify MCP for dependency path analysis
- Linear MCP for ticket creation (Phase 6)

## Procedure

### Phase 1 — Technology & Architecture Recon

1. **Identify the stack and performance-relevant components:**
   ```bash
   # Database
   grep -rn "prisma\|drizzle\|typeorm\|sequelize\|sqlalchemy\|gorm\|diesel" --include="*.ts" --include="*.py" --include="*.go" --include="*.rs" .
   # Caching
   grep -rn "redis\|memcached\|cache\|lru\|memoize" --include="*.ts" --include="*.py" --include="*.go" .
   # Async/Queue
   grep -rn "bull\|bullmq\|celery\|sidekiq\|queue\|worker" --include="*.ts" --include="*.py" --include="*.go" .
   # HTTP client
   grep -rn "axios\|fetch\|got\|requests\|httpclient\|reqwest" --include="*.ts" --include="*.py" --include="*.go" --include="*.rs" .
   ```

2. **Identify the critical paths** — the most frequent and most expensive
   operations:
   - API endpoints (from route definitions)
   - Database queries (from ORM/SQL calls)
   - External API calls (from HTTP client usage)
   - Background jobs (from queue/worker definitions)
   - Rendering paths (frontend components that render frequently)

3. **Record under `## Technology & Architecture` in the report.**

### Phase 2 — Database Performance

4. **N+1 query detection** — the most common and expensive database issue:
   ```bash
   # ORM loops that make per-item queries
   grep -rn "for.*in.*:.*\.find\|for.*in.*:.*\.findOne\|for.*in.*:.*\.get\|\.map.*\.find\|\.map.*\.findOne" --include="*.ts" --include="*.py" --include="*.js" .
   # Missing includes/ joins (load relations separately)
   grep -rn "\.find(\|\.findMany(\|\.findFirst(" --include="*.ts" --include="*.py" . | grep -v "include\|join\|relation"
   ```

5. **Unbounded queries** — queries with no limit:
   ```bash
   grep -rn "\.findMany(\|\.find(\|select.*\*\|all()" --include="*.ts" --include="*.py" --include="*.go" . | grep -v "limit\|take\|pagination\|cursor"
   ```

6. **Missing indexes** — check schema for columns frequently queried but not indexed:
   ```bash
   # Prisma schema
   grep -A5 "@@index\|@@unique" prisma/schema.prisma 2>/dev/null
   # SQL migrations
   grep -rn "CREATE INDEX\|CREATE UNIQUE INDEX" --include="*.sql" .
   # Drizzle
   grep -rn "index(\|unique(" drizzle/ 2>/dev/null
   ```

7. **Transaction misuse** — long-running transactions holding locks:
   ```bash
   grep -rn "transaction\|begin()\|START TRANSACTION" --include="*.ts" --include="*.py" --include="*.go" .
   ```

8. **For each finding:** read the actual code, confirm the issue, estimate the
   query count or data volume, and provide the fix (eager loading, batching,
   pagination, index, or query rewrite).

### Phase 3 — Memory & Resource Management

9. **Memory leaks** — objects that grow without bounds:
   ```bash
   # Global/module-level collections that only add, never remove
   grep -rn "^const.*=.*\[\]\|^const.*=.*Map()\|^const.*=.*Set()\|^const.*=.*{}" --include="*.ts" --include="*.py" --include="*.js" .
   # Event listeners added but never removed
   grep -rn "addEventListener\|on(" --include="*.ts" --include="*.js" . | grep -v "removeEventListener\|off("
   # Set intervals/timeouts not cleared
   grep -rn "setInterval\|setTimeout" --include="*.ts" --include="*.js" . | grep -v "clearInterval\|clearTimeout"
   # Python: module-level dicts/lists that accumulate
   grep -rn "^[a-z_]*\s*=\s*{}\|^[a-z_]*\s*=\s*\[\]" --include="*.py" .
   ```

10. **Resource leaks** — unclosed connections, file handles:
    ```bash
    # Database connections opened but not closed
    grep -rn "createConnection\|connect()\|pool" --include="*.ts" --include="*.py" --include="*.go" . | grep -v "close\|end\|release\|dispose"
    # File handles
    grep -rn "open(\|createReadStream\|createWriteStream" --include="*.ts" --include="*.py" --include="*.go" . | grep -v "close\|end\|dispose\|with "
    ```

11. **For each finding:** confirm the leak by reading the surrounding code, trace
    the lifecycle, and provide the fix (cleanup, dispose, context manager, etc.).

### Phase 4 — Hot Path Analysis

12. **Identify hot paths** — code that runs frequently or under load:
    - API endpoints with high traffic (check analytics or route popularity)
    - Background jobs that run on a schedule
    - Real-time handlers (WebSocket, SSE, polling)
    - Render paths in frontend (components that re-render on every state change)

13. **Check for sync I/O in async paths:**
    ```bash
    # Synchronous file reads in async functions
    grep -rn "readFileSync\|writeFileSync\|execSync\|existsSync" --include="*.ts" --include="*.js" .
    # Python: blocking calls in async functions
    grep -rn "requests\.get\|requests\.post\|time\.sleep\|open(" --include="*.py" . | grep -v "async\|await\|aiohttp"
    ```

14. **Check for unnecessary computation:**
    ```bash
    # Expensive operations in loops
    grep -rn "for.*in.*:.*JSON\.parse\|for.*in.*:.*JSON\.stringify\|for.*in.*:.*regex\|for.*in.*:.*match" --include="*.ts" --include="*.py" --include="*.js" .
    # Redundant re-computation (should be cached/memoized)
    grep -rn "\.filter.*\.filter.*\.map\|\.map.*\.map.*\.filter" --include="*.ts" --include="*.py" --include="*.js" .
    ```

15. **Check for missing pagination:**
    ```bash
    # Endpoints that return lists without pagination
    grep -rn "res\.json(.*findMany\|res\.json(.*findAll\|return.*\.find(" --include="*.ts" --include="*.py" --include="*.js" . | grep -v "limit\|offset\|page\|cursor\|take\|skip"
    ```

16. **For each finding:** trace the full path from entry to exit, estimate the
    cost (CPU, memory, I/O), and provide the optimization.

### Phase 5 — Frontend Performance (if applicable)

17. **Bundle size analysis** (if Node/web frontend):
    ```bash
    # Check for large dependencies
    cat package.json | python3 -c "import json,sys; deps=json.load(sys.stdin).get('dependencies',{}); [print(f'{k}: {v}') for k,v in sorted(deps.items(), key=lambda x: len(x[0]), reverse=True)]"
    # If webpack-bundle-analyzer or similar is configured
    npm run build -- --analyze 2>/dev/null || npx vite-bundle-visualizer 2>/dev/null
    ```

18. **React/render performance** (if React/Vue/Svelte):
    ```bash
    # Re-renders: inline objects/functions as props
    grep -rn "style={{\|onClick={() =>\|renderItem={(\|item={{" --include="*.tsx" --include="*.jsx" .
    # Missing memoization on expensive components
    grep -rn "export.*function\|export const" --include="*.tsx" --include="*.jsx" . | grep -v "memo\|React.memo\|Memo"
    # Missing keys in lists
    grep -rn "\.map(.*=>" --include="*.tsx" --include="*.jsx" . | grep -v "key="
    ```

19. **Asset optimization:**
    ```bash
    # Unoptimized images
    find . -name "*.png" -o -name "*.jpg" -o -name "*.jpeg" 2>/dev/null | grep -v node_modules | grep -v ".min."
    # Large assets
    find . -size +500k -name "*.js" -o -name "*.css" -o -name "*.png" 2>/dev/null | grep -v node_modules
    ```

20. **Web vitals check** (if web app): use `web_search` for "Core Web Vitals
    best practices 2026" and compare against the implementation.

### Phase 6 — Technology-Specific Best Practices

21. **Web search for performance best practices** for each technology:
    Use `web_search` with:
    - "[framework] performance best practices 2026"
    - "[database] query optimization guide"
    - "[ORM] N+1 prevention"
    - "[runtime] memory leak detection"

22. **Compare implementation against best practices:**
    - For each practice: is it applicable? is it implemented?
    - Record as: ✓ Implemented / ✗ Missing / ⚠ Partial / N/A

### Phase 7 — Report and Fix Suggestions

23. **Categorize findings by impact:**

    | Severity | Meaning | Example |
    |---|---|---|
    | CRITICAL | Blocks production or causes OOM/crash | Memory leak that grows unbounded |
    | HIGH | Significant user-visible latency (>1s) | N+1 causing 47 queries per request |
    | MEDIUM | Degraded performance (>200ms) | Missing index on frequent query |
    | LOW | Minor optimization opportunity | Redundant filter chain |

24. **Write fix suggestions with code:**
    - Show the slow code (file:line)
    - Show the optimized code
    - Estimate the improvement (e.g., "47 queries → 2 queries, ~95% reduction")

25. **Write the report** to `.perf-audit/REPORT.md` using `templates/report.md`.

26. **Present in chat:** top 5 findings with estimated impact, and the report path.

### Phase 8 — Create Linear Tickets (optional)

27. Resolve the team: `mcp__linear__get_teams`
28. For each CRITICAL and HIGH finding:
    - Create a Linear issue:
      - **Title:** `perf: [finding]` (e.g., "perf: N+1 query in playtest signup")
      - **Description:** the finding, file:line, estimated impact, the fix
        suggestion with code, and the performance gain estimate
      - **Priority:** 0 (urgent) for CRITICAL, 1 (high) for HIGH
      - **Label:** "performance" if available
29. Confirm ticket URLs back to the user

## Pitfalls

- **Profiling without reading code** — a profiler shows where time is spent, but you need to read the code to know why. Always read the hot path.
- **Optimizing prematurely** — don't optimize code that isn't on a hot path. A 50ms query that runs once a day is fine. The same query in a per-request loop is critical.
- **N+1 false positives** — ORM `.find()` in a loop isn't always N+1. It might be a deliberate per-item lookup with caching. Read the surrounding code.
- **Missing the real bottleneck** — developers optimize the code they can see, but the bottleneck is often in the database (missing index), network (serial API calls), or serialization (large JSON payloads). Look at all layers.
- **Bundle size without tree-shaking analysis** — a large dependency in package.json might be tree-shaken to nothing. Check the actual bundle, not the dependency list.
- **Memory leak false positives** — a growing Map might be a bounded cache with an eviction policy you didn't see. Check for cleanup logic before reporting.
- **Not measuring before and after** — "this should be faster" is not a performance finding. Measure the baseline, apply the fix, measure again. Numbers, not vibes.
- **Frontend-specific issues in a backend project** — if there's no frontend, skip Phase 5 entirely. Don't grep for React patterns in a Go API.
- **Ignoring the database** — the database is usually the bottleneck. Check indexes, query plans, and connection pool config before optimizing application code.

## Verification

- [ ] Technology and architecture recon complete
- [ ] Critical paths identified
- [ ] Database performance audited (N+1, unbounded queries, missing indexes, transactions)
- [ ] Memory and resource management audited (leaks, unclosed connections)
- [ ] Hot path analysis complete (sync I/O, unnecessary computation, missing pagination)
- [ ] Frontend performance audited (if applicable — bundle, renders, assets)
- [ ] Technology-specific best practices retrieved and compared
- [ ] Each finding has file:line, estimated impact, and fix with code
- [ ] Report written to `.perf-audit/REPORT.md`
- [ ] Linear tickets created for CRITICAL/HIGH findings (if user approved)
