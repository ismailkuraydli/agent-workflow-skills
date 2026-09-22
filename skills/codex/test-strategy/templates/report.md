## Test Strategy: [Repo Name]

### Test Inventory

| Type | Count | Files |
|---|---|---|
| Unit | N | [list] |
| Integration | N | [list] |
| E2E | N | [list] |
| Performance | N | [list] |
| **Total** | **N** | |

### Risk Map

| Module | Risk Level | Has Tests? | Coverage | Gap |
|---|---|---|---|---|
| Auth | CRITICAL | unit only | 45% | Missing integration + error path tests |
| Payment | CRITICAL | no | 0% | No tests at all |
| API routes | HIGH | unit + integration | 78% | Missing failure mode tests |
| UI components | LOW | unit | 90% | OK |

### Test Quality Issues

| Issue | Count | Examples |
|---|---|---|
| Tests with no assertions | N | [file:line] |
| Skipped tests | N | [file:line] |
| Excessive mocking | N | [file:line] |
| Snapshot tests | N | [file:line] |

### Coverage Gaps (Critical Paths)

| # | Module | Risk | Missing Test Type | What to Verify | Priority |
|---|---|---|---|---|---|
| 1 | Payment | CRITICAL | All | Transaction integrity, refund flow | Urgent |
| 2 | Auth token | CRITICAL | Error path | Expired/invalid/tampered token | Urgent |
| 3 | DB migration | HIGH | Integration | Rollback, data preservation | High |

### Test Plan

#### Unit Tests
| Test | Module | Verifies | Acceptance Criteria |
|---|---|---|---|
| [name] | [module] | [description] | [pass/fail] |

#### Integration Tests
| Test | Boundary | Verifies | Acceptance Criteria |
|---|---|---|---|
| [name] | [boundary] | [description] | [pass/fail] |

#### E2E Tests
| Test | User Flow | Verifies | Acceptance Criteria |
|---|---|---|---|
| [name] | [flow] | [description] | [pass/fail] |

### Summary
- **Total tests:** N
- **Tests that cannot fail:** N
- **Critical path coverage:** N%
- **Top priority gap:** [module - missing test type]

---
*Strategy by test-strategy skill - [timestamp]*
