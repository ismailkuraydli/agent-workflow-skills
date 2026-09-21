---
name: plan-to-linear
description: Plan bugs/features with TDD, minimal-change, Linear tickets.
---

# Plan-to-Linear

Plan a bug fix or feature end-to-end — context gathering, BDD acceptance criteria, TDD implementation plan, minimal-change challenge — and create a Linear ticket with the full plan document as its description.

## When to Use

- User asks to plan a bug fix or feature before implementing
- User wants a structured plan turned into a Linear ticket
- User asks for acceptance criteria, TDD/BDD plan, or implementation breakdown

## Don't Use For

- Quick fixes that don't warrant a ticket
- Research / spike tasks with no clear deliverable
- Tasks already fully specified in an existing Linear ticket

## Prerequisites

- Linear MCP server configured. If missing: `codex mcp add linear --url https://mcp.linear.app/mcp` (requires rmcp feature enabled)
- A codebase to inspect — use `grep`/`rg` and file reads to verify affected components and reuse opportunities

## Procedure

### Phase 1: Classify & Gather Context

1. Classify the work: **bug**, **feature**, or **chore**. This selects the plan emphasis.
2. Gather context:
   - **Bug:** steps to reproduce, expected vs actual, environment, error output, minimal reproduction
   - **Feature:** user story ("As a __ I want __ so that __"), motivation, links to related issues
3. Record under `## Context / Background` in the plan document (see `templates/plan.md`).

### Phase 2: Scope & Impact

4. Define **in-scope** and **out-of-scope** explicitly. Out-of-scope prevents bloat.
5. Identify **affected components**: use `grep`/`rg` and file reads to find the files, modules, services, APIs that will be touched. Never guess — verify.
6. Assess **risk & impact**: what could break, blast radius, backward compatibility, migration concerns.

### Phase 3: Acceptance Criteria (BDD)

7. Write acceptance criteria in **Given/When/Then** format. Each AC must be testable and binary (pass/fail).
8. Cross-check: every AC traces to the user story or repro. Cut any that don't.

### Phase 4: Test Strategy

9. Define test layers: unit (which modules), integration (which boundaries), e2e (which flows).
10. List edge cases derived from the risk assessment — not generic "test edge cases."

### Phase 5: Implementation Plan (TDD)

11. Break the work into ordered steps following **Red → Green → Refactor**:
    - **Red:** write the failing test first
    - **Green:** minimal code to pass
    - **Refactor:** clean up without changing behavior
12. Each step must be checkable and traceable to an acceptance criterion.
13. Group into sub-tasks if the work spans multiple components.

### Phase 6: Minimal Change Challenge (MANDATORY GATE)

This phase runs after the initial plan is drafted, before the ticket is created. Challenge every planned change and try to eliminate it.

14. **Reuse-first audit:** For each planned new file/function/component, search the codebase with `grep`/`rg` for existing capabilities that already do this — existing utilities, config, framework built-ins, already-imported libraries. If something exists, switch the plan to use it.
15. **Delete-before-add:** Identify dead code, now-redundant branches, or config this change makes obsolete. Lines removed is a positive signal.
16. **Change footprint budget:** Quantify files touched (new vs modified), lines added vs removed. If high for a small feature, rework the plan.
17. **Alternatives considered:** Record at least one heavier approach and why it was rejected. If you can't name one, you haven't looked hard enough.
18. **Scope tightening gate:** Re-read every planned step. If any isn't traceable to an AC, cut it. No speculative abstractions. No "we might need this later."
19. **One-function rule:** Prefer extending an existing function over creating a new one. New abstractions require justification — "cleaner" is not enough; "the existing approach can't handle X without it" is.

**Do not proceed to ticket creation until all items above are satisfied.**

### Phase 7: Create Linear Ticket

20. Resolve the team: call `mcp__linear__get_teams` — find the right team by name or key.
21. Set priority based on severity/impact: 0=urgent, 1=high, 2=medium, 3=low.
22. Create the issue with `mcp__linear__create_issue`:
    - `title`: concise summary
    - `description`: the full plan document (from `templates/plan.md`)
    - `teamId`: resolved team UUID or key
    - `priority`: assessed priority
    - `projectId`: if the feature belongs to a project
23. If sub-tasks warrant separate tracking, create sub-issues via `mcp__linear__create_issue` with `parentId` set to the parent issue ID.
24. Confirm the Linear ticket URL back to the user.

## Pitfalls

- **Skipping the minimal-change challenge** — this is the gate that prevents bloat. If you rush to create the ticket without completing the challenge checklist, the plan is incomplete.
- **Guessing affected components** — always verify with `grep`/`rg` or file reads. Wrong components = wrong plan.
- **Acceptance criteria that aren't testable** — "improves UX" is not an AC. "Given X, when Y, then Z" is.
- **Creating the ticket in the wrong team** — always resolve the team first with `mcp__linear__get_teams`.
- **Empty alternatives section** — if you can't name a heavier approach you rejected, you haven't looked hard enough. Go back to step 17.
- **Template left with placeholders** — every section must be filled. "TBD" means the plan isn't done.

## Verification

- [ ] Plan document has all sections filled (no "TBD" placeholders)
- [ ] Every implementation step traces to an acceptance criterion
- [ ] Minimal Change Challenge checklist is fully checked
- [ ] Linear ticket created — URL returned to user
- [ ] Ticket description matches the plan document exactly
