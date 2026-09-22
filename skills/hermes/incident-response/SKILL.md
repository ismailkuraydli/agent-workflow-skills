---
name: incident-response
description: "Triage incidents, find root cause, write postmortem, create fixes."
version: 0.1.0
author: Ismail Kuraydli, Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [incident, postmortem, root-cause, triage, rollback, runbook]
    related_skills: [release-deploy, code-review, security-audit, linear-ticket]
---

# Incident Response

When production breaks: triage the incident (severity, blast radius), find the
root cause (deploy diff, git bisect, log analysis), stabilize the system
(rollback or fix-forward), write a postmortem (timeline, root cause, contributing
factors, action items), create Linear tickets for fixes, and update the runbook.

The postmortem is the artifact that prevents the same incident from recurring.

## When to Use

- User reports a production issue ("the app is down", "API returning 500s")
- User says "we have an incident" or "production is broken"
- Monitoring alerts fired and the user asks to investigate
- After a deploy caused issues and rollback may be needed
- User asks for a postmortem or RCA (root cause analysis)

## Don't Use For

- Pre-deploy planning → use `release-deploy`
- Security vulnerability discovery → use `security-audit`
- Performance investigation without an active incident → use `performance-audit`

## Prerequisites

- A production system that is experiencing issues (or recently experienced them)
- Access to: git history, `gh` CLI, deployment logs, monitoring dashboards
- Linear MCP for creating fix tickets

## Procedure

### Phase 1 — Triage (first 5 minutes)

1. **Assess severity** — classify the incident:

    | Severity | Meaning | Response | Examples |
    |---|---|---|---|
    | SEV-1 | System down, all users affected | Immediate, all hands | API returns 500 for all requests |
    | SEV-2 | Major feature broken, many users affected | Immediate, on-call | Login broken, data corruption |
    | SEV-3 | Feature degraded, some users affected | During business hours | Slow responses, one endpoint failing |
    | SEV-4 | Minor issue, workaround exists | Next business day | Cosmetic bug, non-critical feature |

2. **Assess blast radius:**
    - Which users are affected? (all, a segment, specific accounts)
    - Which features are broken? (all, a subset)
    - Which regions/environments? (prod only, staging, specific region)
    - Is data at risk? (data loss, data corruption, data exposure)

3. **Check if there was a recent deploy:**
    ```bash
    # Recent deploys
    git log --oneline -20
    gh run list --limit 10
    # Check if a deploy happened recently
    gh deployment list --limit 5 2>/dev/null
    ```

4. **Record the triage summary:**
    - Severity: SEV-X
    - Blast radius: [description]
    - Started at: [timestamp or "unknown"]
    - Detected by: [alert, user report, monitoring]
    - Recent deploy: [yes/no — commit hash, time]

### Phase 2 — Stabilize (stop the bleeding)

5. **If a recent deploy caused the issue — roll back immediately:**
    This is the fastest path to stability. Do not debug before rolling back
    unless the deploy is not reversible.

    ```bash
    # Identify the last known good version
    git log --oneline -10  # find the commit before the issue started

    # Rollback (adapt to deployment mechanism):
    # Kubernetes
    kubectl rollout undo deployment/<name>
    # Heroku
    heroku releases:rollback
    # Docker — redeploy previous image
    docker run <previous-image-tag>
    # Git-based
    git revert <bad-commit> && make deploy
    # CI-based
    gh workflow run deploy.yml --ref <last-good-commit>
    ```

6. **If no recent deploy, or rollback didn't fix it:**
    - Check infrastructure status (cloud provider, DNS, CDN)
    - Check database connectivity and health
    - Check external service status (third-party APIs the system depends on)
    - Check for resource exhaustion (memory, CPU, disk, connections)

    ```bash
    # Health checks
    curl -s https://<production-url>/health
    # Database connectivity
    curl -s https://<production-url>/health/db
    # Check error rates in logs
    # (adapt to logging system: CloudWatch, Datadog, ELK, etc.)
    ```

7. **Verify the system is stable after the action:**
    ```bash
    curl -s -o /dev/null -w "%{http_code}" https://<production-url>/
    # Run smoke tests (same as release-deploy Phase 6)
    ```

8. **If the system cannot be stabilized:** escalate to the user immediately.
    Do not keep trying fixes — the user needs to make a decision about whether
    to involve more people or take manual action.

### Phase 3 — Root Cause Analysis

9. **If a deploy caused the issue — diff the bad deploy:**
    ```bash
    # What changed in the bad deploy
    git diff <last-good-commit>..<bad-commit> --stat
    git diff <last-good-commit>..<bad-commit>
    # Focus on the diff, not the whole codebase
    ```

10. **If the cause is unclear — git bisect:**
    ```bash
    # Find the commit that introduced the bug
    git bisect start
    git bisect bad <bad-commit-or-HEAD>
    git bisect good <last-known-good-commit>
    # For each bisect step: deploy/test/mark
    git bisect run <test-command>  # if automated
    # Or manually: checkout, test, mark good/bad
    git bisect reset  # when done
    ```

11. **Analyze logs for the failure pattern:**
    - What errors appeared and when?
    - Which component failed first?
    - Did the failure cascade to other components?
    - Were there warnings before the failure?

12. **Read the code at the identified commit/line:**
    Use `read_file` and `search_files` to read the actual code that caused the
    issue. Understand not just WHAT broke, but WHY.

13. **Identify contributing factors** — not just the direct cause:
    - Was there a missing test that would have caught this?
    - Was there a code review gap?
    - Was there a configuration change?
    - Was there a dependency update?
    - Was there a data condition that triggered the bug?
    - Was there a race condition or timing issue?

14. **Record the root cause:**
    - Direct cause: [the specific code/config/data that caused the failure]
    - Contributing factors: [list]
    - Why it wasn't caught: [missing test, review gap, etc.]

### Phase 4 — Fix or Mitigate

15. **If the rollback fixed it:** the fix is to correct the bad commit before
    re-deploying. Create a fix branch.

16. **If the issue is still present after rollback:** the problem is in the
    code that was already there before the deploy. Fix-forward:
    - Create a fix branch
    - Write a failing test that reproduces the issue
    - Fix the code
    - Verify the test passes
    - Deploy the fix

17. **Short-term mitigation vs long-term fix:**
    - Mitigation: stop the bleeding (rollback, feature flag off, rate limit,
      circuit breaker)
    - Fix: address the root cause (code fix, test addition, config change)
    - Both are needed — mitigation now, fix in a follow-up PR

### Phase 5 — Postmortem

18. **Write the postmortem document.** This is the most important artifact. It
    must be blameless — focus on the system, not the people.

    Use `templates/postmortem.md`:

    ```markdown
    ## Incident: [Title]

    ### Summary
    [One paragraph: what happened, how long it lasted, who was affected]

    ### Severity: SEV-X

    ### Timeline (all times in [timezone])
    - [time] — [event: first alert, user report, deploy, detection]
    - [time] — [event: triage started, severity assigned]
    - [time] — [event: rollback initiated, mitigation applied]
    - [time] — [event: system stabilized, verified healthy]
    - [time] — [event: root cause identified]
    - [time] — [event: postmortem started]

    ### Impact
    - Users affected: [N or "all" or "N%"]
    - Features affected: [list]
    - Duration: [N minutes/hours]
    - Data impact: [none / N records lost / N records corrupted]

    ### Root Cause
    [What specifically caused the incident. Be technical and specific.]

    ### Contributing Factors
    - [factor 1: e.g., "no test covered this code path"]
    - [factor 2: e.g., "the alert fired 10 minutes after user reports, not before"]
    - [factor 3: e.g., "the deploy happened during high traffic hours"]

    ### What Went Well
    - [thing 1: e.g., "rollback was fast — under 2 minutes"]
    - [thing 2: e.g., "the health check caught the issue before users noticed"]

    ### What Went Poorly
    - [thing 1: e.g., "the deploy wasn't caught by CI because the test was skipped"]
    - [thing 2: e.g., "no alerting on this metric — we found out from users"]

    ### Action Items
    - [ ] [action]: [description] — Owner: [name] — Ticket: [Linear URL]
    - [ ] [action]: [description] — Owner: [name] — Ticket: [Linear URL]

    ### Lessons Learned
    [What this incident taught us about the system, the process, or the team.]
    ```

19. **Write the postmortem to a file:** `.incidents/[date]-[slug].md`

### Phase 6 — Action Items & Linear Tickets

20. **For each action item in the postmortem, create a Linear ticket:**
    - Resolve the team: `mcp__linear__get_teams`
    - Create an issue:
      - **Title:** `fix: [action item description]`
      - **Description:** the action item, the incident reference, the root cause
        it addresses, and the acceptance criteria
      - **Priority:** 0 (urgent) for fixes that prevent recurrence, 1 (high) for
        improvements
      - **Label:** "incident" or "bug" if available
    - Link the ticket in the postmortem action items section

21. **Action item categories:**
    - **Fix:** correct the code that caused the incident
    - **Test:** add a test that would have caught the incident
    - **Monitoring:** add an alert that would have detected it sooner
    - **Process:** change the deploy/review process to prevent recurrence
    - **Documentation:** update the runbook with the new procedure

### Phase 7 — Runbook Update

22. **If the incident revealed a gap in the runbook:**
    - Find the runbook (check `docs/runbook.md`, `RUNBOOK.md`, `ops/runbook/`)
    - Add the new procedure: how to detect this issue, how to mitigate it, how
      to fix it
    - If no runbook exists, create one with this incident as the first entry

### Phase 8 — Report

23. **Report to the user:**
    - Incident severity and status (resolved / mitigated / ongoing)
    - Root cause (one sentence)
    - What was done to stabilize
    - Postmortem file path
    - Linear tickets created (URLs)
    - Action item count and owners

## Pitfalls

- **Debugging before stabilizing** — stop the bleeding first, understand the cause second. Rollback is faster than debugging.
- **Blame in the postmortem** — "Bob deployed bad code" is not a root cause. "The deploy pipeline didn't run the integration test suite" is. Postmortems are blameless.
- **Skipping the postmortem** — "we fixed it, move on" guarantees the same incident happens again. The postmortem is the deliverable.
- **Action items without owners** — "we should add monitoring" with no owner and no ticket means it never happens. Every action item needs an owner and a Linear ticket.
- **Not checking contributing factors** — the direct cause is the trigger, but contributing factors are why the trigger wasn't caught. Fix both.
- **Rolling back without verifying** — after rollback, run smoke tests. A rollback that doesn't fix the issue means the cause is deeper.
- **Fixing forward without a test** — if you fix-forward, write a test that reproduces the issue first. Otherwise you can't verify the fix.
- **Not recording the timeline** — "it broke at some point" is not a timeline. Record specific times for detection, triage, mitigation, and resolution.
- **Ignoring near-misses** — an incident that was caught before users noticed is still an incident. Write a postmortem — the next time you might not be so lucky.

## Verification

- [ ] Incident triaged (severity, blast radius, started time)
- [ ] System stabilized (rollback or fix-forward applied and verified)
- [ ] Root cause identified (direct cause + contributing factors)
- [ ] Postmortem written (blameless, with timeline, impact, root cause, action items)
- [ ] Action items have owners and Linear tickets
- [ ] Runbook updated (if applicable)
- [ ] User received the incident report
