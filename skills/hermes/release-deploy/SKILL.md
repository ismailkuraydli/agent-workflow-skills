---
name: release-deploy
description: "Ship releases: version bump, changelog, deploy, smoke test, rollback."
version: 0.1.0
author: Ismail Kuraydli, Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [release, deploy, semver, changelog, rollback, smoke-test, ci-cd]
    related_skills: [linear-ticket, code-review, repo-health, github]
---

# Release & Deploy

Takes merged work from the default branch to production safely. Generates
release notes from merged PRs, bumps the version (semver), creates a changelog
entry, selects a deployment strategy, runs a pre-deploy checklist, deploys,
runs smoke tests, and prepares a rollback plan. Updates Linear ticket statuses
(In Review → Done) for shipped work.

**This is not CI/CD configuration** — it assumes a deployment pipeline exists
(`gh`, Docker, cloud CLI, or `make deploy`). This skill orchestrates the
release process: what to ship, how to ship it, and how to know it worked.

## When to Use

- User asks to release, deploy, ship, or publish
- User says "let's ship the latest changes"
- After a batch of PRs has been merged and it's time to release
- User asks for release notes or a changelog
- User asks to bump the version
- Before a scheduled deploy window

## Don't Use For

- Implementing a feature (use `plan-to-linear` → `linear-ticket`)
- Reviewing a PR (use `code-review`)
- CI/CD pipeline setup (that's infrastructure work, not a release skill)

## Prerequisites

- A git repository with merged work on the default branch
- `gh` CLI for PR history and GitHub releases
- Deployment mechanism: `make deploy`, Docker push, cloud CLI (`aws`, `gcloud`,
  `fly`, `vercel`, `netlify`, `heroku`), or a deploy script
- Optional: Linear MCP for ticket status updates
- Optional: Graphify MCP for impact analysis of what's being released

## Procedure

### Phase 1 — What's Being Released

1. **Identify the current version** — find the version in:
   ```bash
   cat package.json | grep '"version"' 2>/dev/null
   cat pyproject.toml | grep 'version' 2>/dev/null
   cat Cargo.toml | grep '^version' 2>/dev/null
   cat VERSION 2>/dev/null
   git describe --tags --abbrev=0 2>/dev/null
   ```

2. **Find what changed since the last release:**
   ```bash
   LAST_TAG=$(git describe --tags --abbrev=0 2>/dev/null)
   if [ -z "$LAST_TAG" ]; then
     # No tags — use first commit
     LAST_TAG=$(git rev-list --max-parents=0 HEAD)
   fi
   git log $LAST_TAG..HEAD --oneline
   ```

3. **List merged PRs since last release:**
   ```bash
   gh pr list --state merged --base main --limit 50 --json number,title,labels,body,mergedAt,headRefName
   ```

4. **Categorize each PR:**
   - `feat:` or `feature` label → **Feature** (minor or major bump)
   - `fix:` or `bug` label → **Bug fix** (patch bump)
   - `breaking:` or `breaking-change` label → **Breaking change** (major bump)
   - `chore:`, `refactor:`, `docs:` → **Maintenance** (no bump, changelog only)
   - `security:` or `security` label → **Security fix** (patch or minor bump)
   - `perf:` → **Performance** (patch bump)

5. **Check for unmerged Linear tickets still in "In Review":**
   Call `mcp__linear__search_issues` with a filter for status "In Review" —
   these should NOT be in the release. Flag any that are still open.

### Phase 2 — Version Bump

6. **Determine the bump level** using semver rules:
   - **Major (X.0.0):** any breaking change, or incompatible API modification
   - **Minor (0.X.0):** new features, backward-compatible
   - **Patch (0.0.X):** bug fixes, security fixes, performance improvements
   - **No bump:** chore, refactor, docs only

   When in doubt, minor. Breaking changes should be explicit (a PR labeled
   `breaking:` or a commit message with `BREAKING CHANGE:`).

7. **If this is a pre-1.0 version (0.x.x):**
   - Minor bumps for new features (0.x → 0.x+1)
   - Patch bumps for fixes (0.x.y → 0.x.y+1)
   - Major bumps (→ 1.0.0) only when the API is stable and documented
   - Pre-1.0 means "API may change" — be conservative

8. **Present the proposed version bump to the user:**
   ```
   Current: 1.4.2
   Proposed: 1.5.0
   Reason: 3 features, 0 breaking changes
   ```

9. **Wait for user confirmation.** Do not bump without approval.

10. **Apply the version bump** — update the version in:
    - `package.json` (Node)
    - `pyproject.toml` / `setup.py` (Python)
    - `Cargo.toml` (Rust)
    - `go.mod` (Go — uses tags, no file edit)
    - `VERSION` file (if present)
    - Any other version source (check `git grep -n "1.4.2"`)

### Phase 3 — Release Notes & Changelog

11. **Generate release notes** from the categorized PRs. Format:

    ```markdown
    ## [1.5.0] — 2026-09-21

    ### Features
    - Add player inventory system (#142, #145)
    - Add NPC dialogue generation (#138)

    ### Bug Fixes
    - Fix auth token expiry check using < instead of <= (#143)

    ### Security
    - Patch CVE-2026-1234 in jsonwebtoken (#147)

    ### Performance
    - Batch database queries in playtest signup (#140)

    ### Maintenance
    - Refactor auth middleware into shared module (#144)
    - Update dependencies (#146)
    ```

12. **Append to CHANGELOG.md** (create if it doesn't exist):
    - Follow Keep a Changelog format if one exists
    - If no changelog convention, use the format above
    - Place the new entry at the top, below the header

13. **Create a GitHub release** (if the user wants a public release):
    ```bash
    gh release create v1.5.0 \
      --title "v1.5.0" \
      --notes-file <release-notes-file> \
      --target main
    ```

    For pre-release: add `--prerelease`. For draft: add `--draft`.

### Phase 4 — Pre-Deploy Checklist

14. **Run the pre-deploy checklist.** Each item must pass before proceeding:

    - [ ] All PRs in this release are merged to the default branch
    - [ ] No unmerged branches referenced in release notes
    - [ ] Version bumped in all version files
    - [ ] CHANGELOG.md updated
    - [ ] Git tag created (if using tags)
    - [ ] CI is green on the default branch:
      ```bash
      gh run list --branch main --limit 5 --json status,conclusion,name
      ```
    - [ ] No open Dependabot/security alerts:
      ```bash
      gh api repos/{owner}/{repo}/dependabot/alerts --jq '[.[] | select(.state == "open")] | length'
      ```
    - [ ] Database migrations (if any) are ready and reversible
    - [ ] Feature flags (if any) are configured for the deploy
    - [ ] Environment variables for production are set and verified
    - [ ] Deployment target is healthy (no ongoing incidents)

15. **If any check fails:** stop, report the failure, do not deploy.

### Phase 5 — Deploy

16. **Identify the deployment mechanism** — check for:
    ```bash
    # Common deploy commands
    cat Makefile | grep -i deploy 2>/dev/null
    cat package.json | grep -i '"deploy"' 2>/dev/null
    ls deploy*.sh deploy* 2>/dev/null
    # Cloud CLIs
    which aws gcloud fly vercel netlify heroku 2>/dev/null
    # Docker
    ls Dockerfile docker-compose*.yml 2>/dev/null
    # CI-based deploy
    cat .github/workflows/deploy*.yml 2>/dev/null
    ```

17. **Select deployment strategy** based on what the infrastructure supports:
    - **Rolling** (default for Kubernetes, ECS): replace instances gradually
    - **Blue-green**: deploy to a new environment, switch traffic
    - **Canary**: deploy to a small percentage, monitor, ramp up
    - **Recreate**: stop old, start new (downtime — simple but risky)
    - **Feature flag**: deploy code, enable via flag (no traffic change)

18. **If the deploy is automated via CI:**
    ```bash
    # Trigger the deploy workflow
    gh workflow run deploy.yml --ref main
    # Watch the run
    gh run list --workflow=deploy.yml --limit 1
    gh run watch <run-id>
    ```

19. **If the deploy is manual:**
    ```bash
    # Run the deploy command
    make deploy
    # Or the deploy script
    ./deploy.sh
    # Or push to the deploy branch (Heroku-style)
    git push origin main:production
    ```

20. **Wait for the deploy to complete.** Monitor the output. If it fails:
    - Do NOT retry automatically
    - Report the failure with the error output
    - Proceed to Phase 7 (rollback) if partial deploy occurred

### Phase 6 — Smoke Tests

21. **Run smoke tests against the deployed version.** Smoke tests verify the
    system is alive and serving correctly — not full test suites.

    **Health checks:**
    ```bash
    curl -s https://<production-url>/health | python3 -m json.tool
    curl -s -o /dev/null -w "%{http_code}" https://<production-url>/
    ```

    **Critical path checks** — for each critical user flow, verify it works:
    - Auth: can a user log in?
    - Core feature: can a user do the main action?
    - Data: can a user read/write data?
    - External integrations: are third-party services responding?

    ```bash
    # Example: API health
    curl -s https://api.example.com/health
    # Example: Auth flow
    curl -s -X POST https://api.example.com/auth/login -d '{"email":"test@test.com","password":"test"}' | python3 -m json.tool
    # Example: Database connectivity (via health endpoint)
    curl -s https://api.example.com/health/db
    ```

22. **Compare against pre-deploy baseline:**
    - Response times should be within normal range
    - Error rates should be zero (or within normal range)
    - No new error types in logs

23. **If smoke tests fail:** proceed to Phase 7 (rollback) immediately.

### Phase 7 — Rollback Plan

24. **The rollback plan must be prepared BEFORE the deploy, not after it fails.**
    Prepare it in Phase 4 and execute it here if needed.

25. **Rollback steps** (adapt to the deployment mechanism):
    ```bash
    # Git-based: revert to previous tag
    git checkout v1.4.2
    make deploy

    # Kubernetes: rollback the deployment
    kubectl rollout undo deployment/<app-name>

    # Docker: run the previous image
    docker run <previous-image-tag>

    # Heroku: rollback the release
    heroku releases:rollback

    # AWS ECS: rollback the service
    aws ecs update-service --service <name> --task-definition <previous-revision>
    ```

26. **Database rollback** (if migrations were part of the release):
    - The migration must have a `down`/`revert` function
    - Test the revert before the deploy (Phase 4 checklist)
    - If the migration is NOT reversible: do NOT roll back the DB, roll forward
      with a fix migration instead

27. **If rollback is needed:**
    - Execute the rollback
    - Verify the system is healthy at the previous version
    - Run smoke tests against the rolled-back version
    - Report the incident: what failed, what was rolled back, what needs fixing
    - Create a Linear ticket for the failed release's fix

### Phase 8 — Post-Release

28. **Update Linear ticket statuses:**
    - For each PR in the release, find the linked Linear ticket
    - Call `mcp__linear__search_issues` with the PR number or branch name
    - Move tickets from "In Review" to "Done" (or the team's equivalent)
    - Use `mcp__linear__update_issue` with the resolved status

29. **Post a release announcement** (if the team uses one):
    - Slack/Teams message with the release notes summary
    - Linear comment on the umbrella project (if one exists)
    - GitHub release is already created in Phase 3

30. **Report to the user:**
    - Version released: v1.5.0
    - PRs shipped: N
    - Deploy status: success / failed / rolled back
    - Smoke test results: pass / fail
    - Linear tickets closed: N
    - Any issues encountered
    - Rollback plan status (executed / not needed / prepared)

31. **Schedule the next release** (if using a cadence):
    - If weekly: note the next release date
    - If on-demand: no action needed

## Pitfalls

- **Deploying without a rollback plan** — prepare the rollback in Phase 4, before the deploy. If you can't describe the rollback, you can't deploy.
- **Bumping major without checking breaking changes** — a `fix:` PR that changes a return type IS breaking. Grep the PR diffs for API signature changes before deciding the bump level.
- **Deploying with red CI** — if CI is red on the default branch, stop. A red CI means something is broken, and deploying it makes it worse.
- **Skipping smoke tests** — "the deploy succeeded" ≠ "the system works." A deploy can succeed and the app can be broken (bad env var, missing migration, wrong config).
- **Not testing database migration reversibility** — a migration that can't be reversed is a one-way door. Test the `down`/`revert` before deploying.
- **Including unmerged work in release notes** — only include PRs that are actually merged to the default branch. A PR that's "ready to merge" is NOT in the release.
- **Forgetting to update version in all files** — some projects have the version in multiple places (package.json, setup.py, __init__.py, VERSION). Check `git grep` for the old version after bumping.
- **Retrying a failed deploy automatically** — a failed deploy may have partially applied. Retrying can make it worse. Investigate first.
- **Not closing Linear tickets** — tickets left in "In Review" after the PR is merged and released confuse the team's board. Move them to Done.
- **Deploying during an active incident** — if the system is already having problems, a new deploy makes diagnosis harder. Wait for the incident to resolve.
- **Not monitoring post-deploy** — the first 15 minutes after a deploy are when most issues surface. Watch the logs, metrics, and error rates during this window.

## Verification

- [ ] All merged PRs since last release catalogued and categorized
- [ ] Version bump proposed, confirmed by user, and applied to all version files
- [ ] Release notes generated from PR history
- [ ] CHANGELOG.md updated with new entry
- [ ] GitHub release created (if applicable)
- [ ] Pre-deploy checklist all green (CI, security alerts, migrations, env vars)
- [ ] Deployment mechanism identified and strategy selected
- [ ] Deploy executed and monitored
- [ ] Smoke tests passed against production
- [ ] Rollback plan prepared before deploy (executed if needed)
- [ ] Linear tickets updated (In Review → Done)
- [ ] Post-release report delivered to user
