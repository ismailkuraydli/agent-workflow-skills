## Repo Health: [Repo Name]

### Health Scorecard

| Metric | Value | Trend |
|---|---|---|
| Dead code (unused exports/functions) | N | → |
| Duplication (duplicated blocks) | N | → |
| Max inbound coupling | N (file) | → |
| Circular dependencies | N | → |
| Consistency violations | N | → |
| Tech debt items (total / critical / high) | N | → |
| Avg debt age | N months | → |
| Test coverage (high-risk modules) | N% | → |

### Coverage

- Files scanned: N of M
- Files read in full: N (selected by coupling × complexity × debt density)
- Tools available: [list]
- Tools unavailable: [list — coverage gaps noted]

---

### Lens 1 — Dead Code & Bloat

| # | Severity | File:Line | Finding | Effort | Recommendation |
|---|---|---|---|---|---|
| 1 | HIGH | src/utils/old.ts:42 | Unused export `legacyFormat` | S | Delete + update imports |
| 2 | MEDIUM | package.json | Unused dep `lodash-es` | S | Remove from dependencies |

### Lens 2 — Refactoring Opportunities

| # | Severity | File:Line | Smell | Effort | Refactor |
|---|---|---|---|---|---|
| 1 | HIGH | src/api/handler.ts:120 | Long function (85 lines) | M | Extract `validateInput` and `processRequest` |
| 2 | MEDIUM | src/lib/parser.ts:15 | Duplicated logic (3 copies) | M | Extract shared `parseToken` function |

### Lens 3 — Decoupling Analysis

| # | Severity | Module | Issue | Effort | Suggested Seam |
|---|---|---|---|---|---|
| 1 | HIGH | src/lib/utils.ts | God module (14 inbound deps) | L | Split into `string-utils`, `date-utils`, `validation` |
| 2 | CRITICAL | A→B→C→A | Circular dependency | M | Extract shared module, invert B→A dependency |

### Lens 4 — Code Consistency Audit

| # | Severity | Issue | Files (approach A) | Files (approach B) | Recommendation |
|---|---|---|---|---|---|
| 1 | MEDIUM | Mixed error handling | 8 files: throw | 3 files: return null | Standardize on throw; convert the 3 |
| 2 | LOW | Mixed HTTP client | 12 files: axios | 2 files: fetch | Standardize on axios; convert the 2 |

### Lens 5 — Enhancement Opportunities

| # | Severity | File:Line | Missing | Effort | Recommendation |
|---|---|---|---|---|---|
| 1 | HIGH | src/api/auth.ts:45 | Missing input validation | S | Add zod schema validation |
| 2 | MEDIUM | src/db/queries.ts:80 | N+1 query pattern | M | Batch fetch with `include` |

### Lens 6 — Technical Debt Inventory

| Category | Count | Avg Age | Critical Path | High Churn |
|---|---|---|---|---|
| FIXME (bug debt) | N | N months | N items | N items |
| TODO (design debt) | N | N months | N items | N items |
| HACK/WORKAROUND | N | N months | N items | N items |

**Top debt items (old + critical path):**
1. [file:line] — [text] — [age] — CRITICAL
2. [file:line] — [text] — [age] — HIGH

---

### Recommended Linear Tickets

| Ticket | Findings | Priority | Effort |
|---|---|---|---|
| refactor: extract auth middleware | 4 findings in src/api/ | HIGH | M |
| cleanup: remove dead code in utils | 8 unused exports | MEDIUM | S |
| refactor: break circular dep A→B→C | 1 cycle | CRITICAL | M |

---
*Report by repo-health skill — [timestamp]*
