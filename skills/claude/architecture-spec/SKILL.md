---
name: architecture-spec
description: "Architect systems from requirements to full spec with ADRs."
---
# Architecture Spec

Takes a raw requirement and produces a complete architecture specification
through an iterative question-and-answer process. Asks about the current
system (if one exists), gathers non-functional requirements and constraints,
presents multiple architecture options with trade-off matrices, and iterates
until the user confirms the spec is detailed enough to implement.

The output is a full architecture document with ADRs, component breakdown,
data flow, failure mode analysis, deployment topology, and a migration path —
ready to break into Linear projects and tickets via `plan-to-linear`.

## When to Use

- User asks to design or architect a new system from requirements
- User asks to redesign or evolve an existing system's architecture
- User asks for architecture options or trade-off analysis
- User asks to spec out a system before implementation
- User says "design the architecture for X" or "how should we build X"

## Don't Use For

- Single-feature planning (use `plan-to-linear`)
- Code-level review (use `code-review`)
- Repo health audit (use `repo-health`)
- Understanding an existing codebase (use `codebase-onboarding`)

## Prerequisites

- The requirement (even if vague — the skill will sharpen it)
- If existing system: codebase access via `Grep` and `Read`
- Linear MCP tools for ticket creation (`mcp__linear__get_teams`, `mcp__linear__create_issue`) — only for Phase 7

## Procedure

This is an **interactive skill** — it stops and waits for user input at
multiple gates. Do not skip the questions. The questions are the skill.

### Phase 1 — Requirements Gathering

#### 1.1 Functional Requirements

Ask the user what the system does. If the requirement is vague, ask
clarifying questions ONE AT A TIME (not a wall of 20 questions):

- What are the core use cases? (Who does what?)
- What are the inputs and outputs?
- What triggers the system? (User action, event, schedule, webhook?)
- Are there different user roles or access levels?
- What are the explicit boundaries — what does this NOT do?

Record answers under `## Requirements → Functional Requirements` in the spec
document (see `templates/spec.md`).

#### 1.2 Non-Functional Requirements (NFRs)

This is the step most architects skip and it causes 80% of rework. Ask:

- **Scale:** How many users? How many requests/sec? How much data?
- **Latency:** What response time is acceptable? (p50 vs p99)
- **Throughput:** How many transactions/second? Batch or real-time?
- **Availability:** Uptime target? (99%, 99.9%, 99.99%)
- **Security:** Compliance requirements? (GDPR, HIPAA, SOC2, PCI)
- **Data consistency:** Strong? Eventual? Acceptable staleness window?
- **Read/write ratio:** Mostly reads? Mostly writes? Mixed?
- **Geographic distribution:** Single region? Multi-region? Global?
- **Durability:** How long must data persist? Archival requirements?

Record answers under `## Requirements → Non-Functional Requirements`.

#### 1.3 Constraints Inventory

Ask about hard constraints that eliminate options:

- **Team:** How many engineers? What skills/stacks do they know?
- **Budget:** Monthly infra budget? (rough range is fine)
- **Timeline:** When does this need to ship? MVP vs full system?
- **Existing infrastructure:** What cloud/provider are you on? What's already
  running that you must integrate with?
- **Org policies:** Approved tech list? Mandated tools? Network restrictions?
- **Compliance:** Regulatory requirements that constrain data location,
  encryption, access logging?

Record answers under `## Requirements → Constraints`.

### Phase 2 — Current State Audit (if applicable)

If an existing system exists, ask:

- What architecture is currently in place? (If codebase available, verify with
  `Grep` and `Read` — don't trust a verbal description alone)
- What works well that you want to keep?
- What are the pain points? (Performance, scalability, maintainability,
  deployment, coupling — `repo-health` can feed this)
- What is off-limits or too risky to change?
- What tech debt constrains new choices?

If a codebase is available, run a quick `codebase-onboarding` pass:
- Identify entry points, frameworks, data layer, external integrations
- Map the current component graph
- Identify trust boundaries

Record under `## Current State`.

If no existing system: skip this phase, note "greenfield" in the spec.

### Phase 3 — Architecture Options

Present **2-4 architecture options**. Not one — the user picks. Each option must
be genuinely different in approach, not just a tweak:

#### 3.1 Generate options

For each option:
- **Name** — short, descriptive (e.g., "Monolithic API + Background Workers",
  "Event-Driven Microservices", "Modular Monolith with CQRS")
- **Description** — 2-3 sentences: how it works, what it optimizes for
- **Architecture diagram** — ASCII or text-based component diagram
- **Component list** — what services/modules exist and what each does
- **Technology suggestions** — concrete stack per component (not just "a
  database" — "PostgreSQL 16 with read replicas")
- **Trade-off matrix** — scored against the NFRs from Phase 1.2:

| NFR | Option A | Option B | Option C |
|---|---|---|---|
| Scale (req/sec) | 10k ✓ | 50k ✓✓ | 100k ✓✓✓ |
| Latency (p99) | <100ms ✓ | <50ms ✓✓ | <200ms ✗ |
| Availability | 99.9% ✓ | 99.95% ✓✓ | 99.99% ✓✓ |
| Team fit | ✓✓✓ (2 devs) | ✗ (needs 5) | ✗✗ (needs 8) |
| Monthly cost | $200 ✓✓✓ | $2k ✓✓ | $8k ✗ |
| Time to ship | 2 weeks ✓✓✓ | 2 months ✗ | 4 months ✗✗ |

- **Cost estimate** — rough monthly infra cost (compute + storage + network +
  managed services). This kills options early and saves weeks of deliberation.
- **Risk assessment** — what's the riskiest assumption? What's unknown? What
  could fail?
- **Spike candidate** — can we validate the riskiest assumption in a day?

#### 3.2 Present options

Present the options to the user and **wait for them to pick**. Do not
recommend — the user picks based on their constraints, which you may not fully
know. Ask: "Which option do you want to go with, or would you like a hybrid of
any of these?"

If the user asks for a hybrid, synthesize a new option from the pieces and
present it with the same trade-off matrix.

### Phase 4 — Deepen the Selected Architecture

Once the user picks an option, deepen it iteratively. For each area below,
present what you have and ask: "Is this detailed enough, or should I go deeper?"

#### 4.1 Component Breakdown

For each component:
- **Responsibility** — what it does, in one sentence
- **Inputs** — what it receives (API contracts, events, file types)
- **Outputs** — what it produces
- **Dependencies** — what it depends on (upstream) and what depends on it
  (downstream)
- **Technology** — concrete stack and version
- **Scaling** — how it scales (horizontal, vertical, sharding, partitioning)
- **State** — stateless? stateful? what state does it hold?

#### 4.2 Interface Contracts

For each component boundary:
- **API contract** — endpoints, request/response shapes, error codes, auth
- **Event contracts** — event names, payload schema, ordering guarantees
- **Data contracts** — shared data models, schema, ownership (who writes, who
  reads)

If the user wants to go deeper, specify the contract for each endpoint/event
in full.

#### 4.3 Data Flow

- **Write path** — how a write enters, flows through components, and persists
- **Read path** — how a read is served (direct, cache, materialized view)
- **Cache strategy** — what is cached, where, invalidation approach
- **Async paths** — what goes through queues, what is processed synchronously
- **Data stores** — which database for which data, replication topology

#### 4.4 Failure Mode Analysis

For each component:
- What happens when it fails? (Single point of failure? Degradation? Outage?)
- What happens when a dependency fails?
- What is the blast radius?
- What is the recovery mechanism? (Restart, failover, replay, manual)
- What is silent vs noisy? (Does the system degrade silently or alert?)

Identify:
- Single points of failure (SPOFs)
- Cascading failure risks (one component's failure triggers others)
- Data loss scenarios
- Silent failure paths (system looks healthy but isn't)

#### 4.5 Deployment Topology

- **Environments** — dev, staging, prod — how do they differ?
- **Scaling** — auto-scaling groups, instance counts, scaling triggers
- **Regions** — single region? multi-region active-active? active-passive?
- **Network** — VPC topology, subnets, security groups, CDN, DNS
- **CI/CD** — how does code get from commit to prod? Blue-green, canary, rolling?
- **Observability** — metrics, logging, tracing, alerting — what stack?

#### 4.6 Migration Path (if existing system)

- How do we get from current → target without downtime?
- **Strangler fig** — incrementally replace pieces of the old system
- **Big bang** — cutover at a maintenance window (risky, fast)
- **Feature flag** — deploy new system behind a flag, switch over gradually
- **Parallel run** — run both systems side by side, compare outputs
- **Data migration** — how to migrate data without loss or inconsistency
- **Rollback plan** — if the migration fails, how to revert

#### 4.7 Iterative Deepening Protocol

After completing 4.1-4.6, present the full spec to the user and ask:

"The spec currently covers [list of sections]. For each section, is it detailed
enough to implement, or should I go deeper?"

For any section the user says "go deeper":
- Re-enter that section
- Ask specific follow-up questions
- Add detail until the user confirms

Repeat until the user says the spec is complete. Do not decide for them — they
know their team and constraints.

### Phase 5 — Architecture Decision Records (ADRs)

For each major decision in the spec, write an ADR:

```
### ADR-001: [Decision title]

**Context:** Why this decision is needed — the constraint or problem.
**Options considered:** What alternatives were evaluated (from Phase 3 options
  + any sub-decisions made in Phase 4).
**Decision:** What was decided.
**Rationale:** Why this option was chosen over the others.
**Consequences:** What this decision enables, what it precludes, and what to
  watch for in the future.
```

ADRs go under `## Architecture Decision Records` in the spec. Number them
sequentially. These survive team turnover — they're the "why" behind the "what".

### Phase 6 — Finalize Spec

Compile the complete spec document using `templates/spec.md`:

1. Requirements (functional, non-functional, constraints)
2. Current state (if applicable)
3. Selected architecture (with rationale)
4. Component breakdown
5. Interface contracts
6. Data flow
7. Failure mode analysis
8. Deployment topology
9. Migration path (if applicable)
10. Architecture Decision Records
11. Open questions / deferred decisions
12. Implementation plan (Linear breakdown)

Write it to a file (e.g., `ARCHITECTURE.md` or `<project>-architecture.md`)
and present the path to the user.

#### Open Questions

Explicitly list decisions that were deferred — things you genuinely can't
decide yet because they depend on unknowns (spike results, team availability,
budget approval). These are tracked, not hidden:

```
- [ ] OQ-1: Will the queue sustain 10k msg/sec? Spike needed.
- [ ] OQ-2: Multi-region active-active or active-passive? Depends on budget.
- [ ] OQ-3: Do we need a separate analytics DB? Depends on query patterns.
```

### Phase 7 — Create Linear Projects and Tickets (optional)

If the user wants to move to implementation:

1. Resolve the team: `mcp__linear__get_teams`
2. Create a **Linear project** for the overall architecture (if the workspace
   supports projects via `mcp__linear__create_issue` or `linear_ops_run`)
3. Break the architecture into **phases** — ordered implementation stages
4. For each component or phase, create a Linear issue:
   - **Title:** `[phase] Component: [name]` (e.g., "Phase 1: Auth Service")
   - **Description:** the component's responsibility, interfaces, dependencies,
     and acceptance criteria from the spec
   - **Priority:** based on phase ordering (phase 1 = urgent, later = medium)
   - **Project:** linked to the architecture project
5. Create sub-issues for each interface contract within a component
6. Identify **spike candidates** — high-risk unknowns that need a throwaway
   prototype before commitment. Create spike tickets labeled "spike".
7. Confirm ticket URLs back to the user

**Phase ordering rules:**
- Foundational components first (data stores, auth, core domain)
- Components with no downstream dependencies before those that depend on them
- Spikes before their dependent components
- Migration steps in migration order (if applicable)

## Pitfalls

- **Skipping the NFR questions** — this is the #1 cause of architecture rework.
  "It needs to scale" is not an NFR. "10k requests/sec, p99 < 100ms, 99.9%
  uptime" is. Get specific numbers.
- **Presenting only one option** — the user picks, not you. If you only present
  one, you've made the decision for them. Always present 2-4 genuinely
  different options.
- **Recommending an option** — present the trade-off matrix and let the user
  decide. You may not know their budget or team constraints fully. If they ask
  "which would you pick?", answer with the trade-offs, not a recommendation.
- **Not verifying the current system** — if a codebase exists, use `Grep`
  and `Read` to verify the architecture. A verbal description is often
  wrong or incomplete.
- **Options that are tweaks, not alternatives** — "Postgres on AWS" and
  "Postgres on GCP" are not different architectures. Different approaches to
  the problem are (monolith vs microservices, sync vs event-driven, SQL vs NoSQL).
- **Ignoring the cost estimate** — an architecture that costs $8k/month when the
  budget is $500 is dead on arrival. Put costs in the trade-off matrix.
- **Deepening everything to the same level** — the user decides where to go
  deep. Don't spec the CDN DNS configuration if they care about the data model.
  Follow their lead.
- **Hiding deferred decisions** — if you can't decide something, say so. An
  explicit open question is better than a silent assumption that breaks later.
- **Writing the spec without iterating** — this is an interactive skill. The
  questions and the user's answers shape the architecture. Don't generate the
  whole spec in one pass and present it as final.
- **ADR without consequences** — the "Consequences" field is what makes an ADR
  useful. "Enables X, precludes Y, watch for Z" is the format. Skip it and the
  ADR is just a decision log nobody reads.

## Verification

- [ ] Functional requirements gathered and recorded
- [ ] NFRs gathered with specific numbers (not "scalable" or "fast")
- [ ] Constraints inventory complete (team, budget, timeline, infra, policies)
- [ ] Current state audited (if applicable) — verified against codebase
- [ ] 2-4 genuinely different architecture options presented with trade-off matrices
- [ ] User selected an option (or hybrid)
- [ ] Component breakdown complete for selected architecture
- [ ] Interface contracts specified (API, events, data)
- [ ] Data flow mapped (read path, write path, cache, async)
- [ ] Failure mode analysis complete (SPOFs, cascading, data loss, silent)
- [ ] Deployment topology specified
- [ ] Migration path documented (if applicable)
- [ ] ADRs written for each major decision
- [ ] Open questions listed explicitly
- [ ] User confirmed spec is detailed enough to implement
- [ ] Spec written to file
- [ ] Linear projects/tickets created (if user approved)
