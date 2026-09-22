## Dependency Upgrade Report: [Repo Name]

### Outdated Summary

| Risk | Count | Packages |
|---|---|---|
| CRITICAL (security) | N | [list] |
| HIGH (major) | N | [list] |
| MEDIUM (minor) | N | [list] |
| LOW (patch) | N | [list] |
| **Total** | **N** | |

### Upgrades Performed

| Package | From | To | Risk | Breaking? | Tests | PR |
|---|---|---|---|---|---|---|
| [name] | 1.4.2 | 1.4.3 | LOW | No | Pass | [#N] |
| [name] | 2.1.0 | 3.0.0 | HIGH | Yes — renamed API | Pass (fixed 2 call sites) | [#N] |

### Skipped (Needs Migration)

| Package | From | To | Risk | Why Skipped | Ticket |
|---|---|---|---|---|---|
| [name] | 1.0.0 | 2.0.0 | HIGH | Requires config rewrite, 15 call sites affected | [Linear URL] |

### Test Results

| Metric | Before | After | Delta |
|---|---|---|---|
| Total tests | N | N | 0 |
| Passed | N | N | 0 |
| Failed | N | N | 0 |
| Skipped | N | N | 0 |
| Lint errors | N | N | 0 |

### PRs Created
- [#N] deps: upgrade [package] [old] → [new]
- [#N] deps: batch upgrade patch versions

---
*Report by dep-upgrade skill — [timestamp]*
