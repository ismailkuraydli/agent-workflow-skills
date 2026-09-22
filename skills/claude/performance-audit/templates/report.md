## Performance Audit: [Repo Name]

### Technology & Architecture
| Layer | Technology | Version |
|---|---|---|
| Runtime | [name] | [version] |
| Framework | [name] | [version] |
| Database | [name] | [version] |
| ORM | [name] | [version] |
| Cache | [name] | [version] |

### Critical Paths Identified
1. [path — why it's critical]
2. [path — why it's critical]

---

### Findings by Severity

#### CRITICAL
| # | Finding | File:Line | Impact | Fix | Estimated Gain |
|---|---|---|---|---|---|
| 1 | Memory leak: event listeners never removed | src/app.ts:45 | OOM after ~24h | Add cleanup in useEffect | Prevents crash |

#### HIGH
| # | Finding | File:Line | Impact | Fix | Estimated Gain |
|---|---|---|---|---|---|
| 1 | N+1 query in playtest signup | src/db/signup.ts:80 | 47 queries per request | Eager load with include | 47→2 queries (~95%) |

#### MEDIUM
| # | Finding | File:Line | Impact | Fix | Estimated Gain |
|---|---|---|---|---|---|
| 1 | Missing index on email lookup | migrations/001.sql | Slow login (~400ms) | Add index on users.email | ~400ms→<10ms |

#### LOW
| # | Finding | File:Line | Fix |
|---|---|---|---|
| 1 | Redundant filter chain | src/utils.ts:20 | Combine into single pass |

---

### Best Practices Gap Analysis
| Practice | Status | Recommendation |
|---|---|---|
| Connection pooling | ✓ | Configured correctly |
| Query pagination | ⚠ | Applied to most endpoints, missing on /api/all |
| Response caching | ✗ | No cache headers on static assets |
| Bundle splitting | ✓ | Code splitting implemented |
| Image optimization | ✗ | No next/image or lazy loading |

---

### Summary
- **CRITICAL:** N findings
- **HIGH:** N findings
- **MEDIUM:** N findings
- **LOW:** N findings
- **Estimated total improvement:** [e.g., "47 queries → 2, 400ms → 10ms, prevents OOM"]

---
*Audit by performance-audit skill — [timestamp]*
