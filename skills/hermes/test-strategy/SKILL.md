---
name: test-strategy
description: "Map tests to risk, find coverage gaps, design test pyramid per feature."
version: 0.1.0
author: Ismail Kuraydli, Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [testing, test-strategy, coverage, test-pyramid, risk-mapping, quality]
    related_skills: [code-review, linear-ticket, plan-to-linear, repo-health]
---

# Test Strategy

Ensures the right tests exist, not just any tests. Maps test coverage against
risk, identifies tests that cannot fail, designs the test pyramid for a feature,
and creates a test plan with acceptance criteria.

## When to Use

- User asks "are we testing the right things?"
- User asks for a test plan or test strategy
- User asks "what tests are missing?"
- Before implementing a feature (design the tests first)
- After repo-health reveals coverage gaps in critical paths
- Periodic test suite audit

## Don't Use For

- Writing individual tests (use linear-ticket Phase 5 TDD)
- Running tests (use terminal with the project test command)
- Code review (use code-review lane 3 for TDD discipline check)

## Prerequisites

- A codebase with (or without) tests
- Access to test files via search_files and read_file
- Optional: coverage tools (pytest-cov, jest --coverage, go test -cover)
- Linear MCP for creating test tickets

## Procedure

### Phase 1 - Test Inventory

1. Find all existing tests:
   ```bash
   find . -name "*.test.*" -o -name "*.spec.*" 2>/dev/null | grep -v node_modules
   find . -name "test_*.py" -o -name "*_test.py" 2>/dev/null | grep -v __pycache__
   find . -name "*_test.go" 2>/dev/null
   find . -name "*.gut.*" -o -path "*/tests/*" -name "*.gd" 2>/dev/null
   ```

2. Categorize each test by type: Unit, Integration, E2E, Performance, Contract, Snapshot.

3. Count tests per type and per module. Record under Test Inventory.

### Phase 2 - Risk Mapping

4. Identify critical paths: auth, payment, data persistence, external APIs,
   security-sensitive operations, core business logic.

5. Rate each module: CRITICAL (data loss, breach, financial), HIGH (feature
   broken for many), MEDIUM (degraded for some), LOW (cosmetic).

6. Map tests to risk: for each critical/high module, check if tests exist, what
   types, what coverage. Record gaps: CRITICAL module with no tests = top priority.

### Phase 3 - Test Quality Audit

7. Find tests that cannot fail (no assertions, skipped, excessive mocking,
   snapshot-only):
   ```bash
   grep -rn "def test_.*:" --include="*.py" . | grep -v "assert"
   grep -rn "skip\|pending\|xit\|xdescribe\|@pytest.mark.skip" --include="*.py" --include="*.ts" .
   grep -rn "toMatchSnapshot\|snapshot" --include="*.ts" --include="*.js" .
   grep -rn "mock\|Mock\|stub\|patch\|jest.fn" --include="*.ts" --include="*.py" .
   ```

8. For each finding: read the test, confirm the issue, recommend fix (add
   assertions, un-skip, reduce mocking, replace snapshots with explicit assertions).

### Phase 4 - Coverage Gap Analysis

9. Run coverage if tooling available:
   ```bash
   pytest --cov=src --cov-report=term-missing 2>/dev/null
   jest --coverage --collectCoverageFrom='src/**' 2>/dev/null
   go test -cover -coverprofile=coverage.out ./... 2>/dev/null
   ```

10. Identify uncovered critical paths from the risk map. Modules with 0%
    coverage on critical paths = top priority gap.

11. Identify missing test types: only unit? missing integration/e2e. Only happy
    path? missing error/edge case tests. No failure mode tests (DB down, API timeout)?

### Phase 5 - Test Plan Design

12. Design the test pyramid: many unit tests (fast, isolated), some integration
    tests (cross-component), few E2E tests (full system, slow).

13. For each layer specify: which functions/modules, what edge cases, what error
    paths, what acceptance criteria.

14. Write the test plan with acceptance criteria for each test.

### Phase 6 - Report and Linear Tickets

15. Write the report to .test-strategy/REPORT.md using templates/report.md.

16. Create Linear tickets for CRITICAL/HIGH coverage gaps:
    - Title: test: [module] - [missing test type]
    - Description: module, risk, missing test, what it should verify, acceptance criteria
    - Priority: 0 for CRITICAL, 1 for HIGH
    - Label: testing if available

17. Present in chat: test inventory summary, top 5 coverage gaps, tests that
    cannot fail count, report path.

## Pitfalls

- Equating coverage with quality: 100% coverage with tests that cannot fail is worse than 60% with real verification.
- Testing the mocks: if a test mocks everything, it tests the mock, not the code.
- Snapshot tests as coverage: false confidence, a changed snapshot does not tell you if the change is correct.
- Only testing happy paths: the most important tests verify what happens when things go wrong.
- Missing failure mode tests: what happens when DB is down, API times out, disk is full.
- Inverted test pyramid: too many E2E, too few unit. E2E are slow, fragile, hard to debug.
- Skipped tests as temporary: a skipped test does not exist. Delete or un-skip.
- Not testing the right layer: test at the lowest layer that can verify the behavior.
- Ignoring test speed: a suite that takes 10 minutes does not get run.

## Verification

- [ ] Test inventory complete (all test files found, categorized by type)
- [ ] Risk mapping complete (all modules rated, tests mapped to risk)
- [ ] Test quality audit complete (cannot-fail tests, excessive mocking, snapshots)
- [ ] Coverage gap analysis complete (critical paths with missing tests identified)
- [ ] Missing test types identified (unit/integration/e2e/performance/failure)
- [ ] Test plan designed with acceptance criteria per test
- [ ] Report written to .test-strategy/REPORT.md
- [ ] Linear tickets created for CRITICAL/HIGH gaps (if user approved)
