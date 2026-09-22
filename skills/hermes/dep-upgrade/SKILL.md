---
name: dep-upgrade
description: "Scan outdated deps, assess breaking changes, upgrade with tests."
version: 0.1.0
author: Ismail Kuraydli, Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [dependencies, upgrade, outdated, breaking-changes, security, supply-chain]
    related_skills: [security-audit, code-review, release-deploy, test-strategy]
---

# Dependency Upgrade

Scans for outdated dependencies, assesses breaking changes from changelogs,
creates a test plan for each upgrade, upgrades one at a time with verification,
and creates a PR per upgrade or batch by risk level. Turns "nobody updates
deps until something breaks" into a structured, safe process.

## When to Use

- User asks to update or upgrade dependencies
- User says "are my dependencies outdated?"
- User asks to check for dependency updates
- After `security-audit` finds vulnerable dependencies
- Periodic dependency maintenance (monthly)
- User says "update npm packages" or "upgrade pip packages"

## Don't Use For

- Security vulnerability investigation (use `security-audit` — it finds CVEs, this upgrades deps)
- Release process (use `release-deploy` — but a dep upgrade may trigger a release)

## Prerequisites

- A git repository with a package manifest (package.json, requirements.txt, Cargo.toml, go.mod, etc.)
- `gh` CLI for creating PRs
- Optional: `npm-check-updates`, `pip-audit`, `cargo-outdated`, `go list -m -u all`
- Linear MCP for tracking upgrade tickets

## Procedure

### Phase 1 — Scan for Outdated Dependencies

1. **Run the package manager's outdated checker:**
   ```bash
   # Node
   npm outdated --json 2>/dev/null || pnpm outdated --json 2>/dev/null || yarn outdated --json 2>/dev/null
   # Python
   pip list --outdated --format=json 2>/dev/null || pip list --outdated 2>/dev/null
   # Rust
   cargo outdated 2>/dev/null || cargo update --dry-run 2>/dev/null
   # Go
   go list -m -u all 2>/dev/null
   # Ruby
   bundle outdated 2>/dev/null
   ```

2. **Also check for security advisories:**
   ```bash
   npm audit --json 2>/dev/null
   pip-audit --format json 2>/dev/null
   cargo audit --json 2>/dev/null
   govulncheck ./... 2>/dev/null
   ```

3. **For each outdated dependency, record:**
   - Package name
   - Current version
   - Latest version
   - Version difference: patch / minor / major
   - Security advisory? (from step 2)
   - Direct dependency or transitive?

### Phase 2 — Risk Assessment

4. **Classify each upgrade by risk:**

   | Risk | Criteria | Examples |
   |---|---|---|
   | LOW | Patch bump, no breaking changes | `1.4.2` → `1.4.3` |
   | MEDIUM | Minor bump, backward compatible | `1.4.2` → `1.5.0` |
   | HIGH | Major bump, breaking changes | `1.4.2` → `2.0.0` |
   | CRITICAL | Security advisory + outdated | CVE in current version |

5. **For each HIGH risk upgrade, fetch the changelog:**
   Use `web_search` to find the package's changelog:
   - "[package name] changelog [target version]"
   - "[package name] breaking changes [target version]"
   - "[package name] migration guide [target version]"

6. **Assess breaking changes:**
   - Read the changelog/migration guide
   - Identify which breaking changes affect THIS codebase
   - Use `search_files` to find usage of the changed APIs:
     ```bash
     grep -rn "import.*[package]" --include="*.ts" --include="*.py" --include="*.go" --include="*.rs" .
     grep -rn "from.*[package]" --include="*.ts" --include="*.py" --include="*.go" --include="*.rs" .
     ```
   - Record: which APIs are used, which are affected by breaking changes, what needs to change

7. **Batch upgrades by risk:**
   - **Batch 1 (LOW risk):** all patch bumps together — one PR
   - **Batch 2 (MEDIUM risk):** minor bumps, one PR per package or small group
   - **Batch 3 (HIGH risk):** major bumps, one PR per package, full migration
   - **Batch 0 (CRITICAL):** security upgrades — do first, one PR per package

### Phase 3 — Pre-Upgrade Baseline

8. **Record the test baseline BEFORE upgrading:**
   ```bash
   # Run tests and capture results
   npm test 2>&1 | tee /tmp/dep-upgrade-baseline.log
   # Python
   pytest --tb=short 2>&1 | tee /tmp/dep-upgrade-baseline.log
   # Go
   go test ./... 2>&1 | tee /tmp/dep-upgrade-baseline.log
   ```

9. **Record the baseline:**
   - Total tests, passed, failed, skipped
   - Lint results
   - Build status
   - Any pre-existing failures (these are NOT caused by the upgrade)

### Phase 4 — Upgrade and Verify

10. **For each batch, starting with CRITICAL → LOW:**

    **Upgrade one package (or one batch):**
    ```bash
    # Node
    npm install package@latest
    # Python
    pip install --upgrade package
    # Rust
    cargo update -p package
    # Go
    go get package@latest
    ```

11. **Run the test suite immediately after upgrading:**
    ```bash
    npm test 2>&1 | tee /tmp/dep-upgrade-test.log
    ```

12. **Compare against baseline:**
    - Did any NEW failures appear? (failures not in the baseline)
    - Did any previously-passing tests start failing?
    - Did the test count change? (fewer tests = something broke silently)
    - Are there new lint errors?

13. **If tests fail:**
    - Read the failure output
    - Check if it's a breaking change from the changelog (expected)
    - Fix the code to work with the new API
    - Re-run tests
    - If the fix is too complex: revert the upgrade, create a Linear ticket for a dedicated migration

14. **If tests pass:**
    - Run lint and typecheck
    - Run the build
    - If everything passes: commit the upgrade

15. **Commit message format:**
    ```
    deps: upgrade [package] from [old] to [new]

    [Changelog reference or breaking change notes if applicable]
    ```

### Phase 5 — PR Creation

16. **For each batch, create a PR:**
    ```bash
    git checkout -b deps/upgrade-[package-or-batch]
    git push -u origin deps/upgrade-[package-or-batch]
    gh pr create --title "deps: upgrade [package] [old] → [new]" --body "[description]"
    ```

17. **PR body should include:**
    - Package name and version change
    - Risk level (LOW/MEDIUM/HIGH/CRITICAL)
    - Breaking changes (if any) and how they were addressed
    - Test results (passed/failed vs baseline)
    - Changelog link
    - Security advisory reference (if applicable)

### Phase 6 — Report and Linear Tickets

18. **Write the report:**
    - Total outdated: N
    - By risk: CRITICAL N, HIGH N, MEDIUM N, LOW N
    - Upgraded: N (which, from what to what)
    - Skipped: N (why — too complex, needs dedicated migration)
    - PRs created: N (URLs)
    - Test results: before vs after

19. **Create Linear tickets for skipped HIGH-risk upgrades:**
    - Title: `deps: migrate [package] [old] → [new]`
    - Description: what needs to change, which APIs are affected, migration guide link
    - Priority: 1 (high) for security-motivated, 2 (medium) for others
    - Label: "tech-debt" or "dependencies"

## Pitfalls

- **Upgrading everything at once** — if 5 packages break, you can't tell which one caused it. One batch at a time, starting with lowest risk.
- **Not recording the baseline** — "tests pass after upgrade" means nothing if they passed before too. You need to know if NEW failures appeared.
- **Skipping the changelog** — a major version bump without reading the migration guide is a guaranteed breakage. Always check breaking changes.
- **Ignoring transitive dependencies** — a transitive dep upgrade can break a direct dep. Run the full test suite, not just the package's tests.
- **Upgrading without a worktree** — upgrading on the main branch pollutes it. Use a branch or worktree.
- **Not checking for lockfile changes** — some upgrades change the lockfile significantly. Review the lockfile diff for unexpected transitive changes.
- **Trusting "backward compatible" minor bumps** — a minor bump can still break things in practice (changed defaults, removed deprecated APIs). Always test.
- **Not creating PRs for upgrades** — committing directly to main skips code review. Always create a PR, even for patch bumps.
- **Forgetting to update the lockfile** — upgrading the manifest without updating the lockfile means the next CI run reinstalls the old version. Commit the lockfile.

## Verification

- [ ] All outdated dependencies scanned and recorded
- [ ] Security advisories checked (npm audit / pip-audit / cargo audit)
- [ ] Each upgrade classified by risk (LOW/MEDIUM/HIGH/CRITICAL)
- [ ] Changelogs read for HIGH risk upgrades
- [ ] Breaking changes assessed against actual codebase usage
- [ ] Test baseline recorded BEFORE any upgrades
- [ ] Upgrades done one batch at a time (CRITICAL → LOW)
- [ ] Each upgrade verified against baseline (no new failures)
- [ ] Lint and typecheck pass after each upgrade
- [ ] PR created for each batch with full description
- [ ] Report written with before/after comparison
- [ ] Linear tickets created for skipped HIGH-risk migrations
