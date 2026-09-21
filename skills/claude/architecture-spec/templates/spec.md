# Architecture Spec: [Project Name]

## Requirements

### Functional Requirements
- **Core use cases:** [who does what]
- **Inputs:** [what enters the system]
- **Outputs:** [what the system produces]
- **Triggers:** [user action, event, schedule, webhook]
- **Roles:** [user roles and access levels]
- **Boundaries:** [what this system does NOT do]

### Non-Functional Requirements

| NFR | Target | Notes |
|---|---|---|
| Scale (req/sec) | N | [peak vs average] |
| Latency (p99) | < Nms | [which operations] |
| Throughput | N/sec | [batch or real-time] |
| Availability | N% | [uptime SLO] |
| Security/Compliance | [GDPR/HIPAA/SOC2/PCI] | [scope] |
| Data consistency | [strong/eventual] | [staleness window] |
| Read/write ratio | N:N | [mostly reads? writes?] |
| Geographic | [single/multi-region] | [which regions] |
| Data volume | N TB/GB | [growth rate] |
| Durability | [N years] | [archival needs] |

### Constraints
- **Team:** N engineers, skills: [stacks]
- **Budget:** ~$N/month infra
- **Timeline:** MVP by [date], full system by [date]
- **Existing infrastructure:** [cloud, services, integrations]
- **Org policies:** [approved tech, mandated tools, restrictions]
- **Compliance constraints:** [data location, encryption, access logging]

## Current State (if applicable)

### Architecture Map
[current component graph — verified against codebase]

### What Works
- [keep these]

### Pain Points
- [fix these]

### Off-Limits
- [do not touch these]

## Options Considered

### Option A: [Name]
**Description:** [2-3 sentences]
**Component diagram:** [ASCII/text diagram]
**Components:**
- Component 1: [responsibility, tech]
- Component 2: ...

**Trade-off matrix:**

| NFR | Option A | Option B | Option C |
|---|---|---|---|
| Scale | ✓ | ✓✓ | ✓✓✓ |
| Latency | ✓✓ | ✓ | ✗ |
| Availability | ✓ | ✓✓ | ✓✓ |
| Team fit | ✓✓✓ | ✗ | ✗✗ |
| Cost/month | $N | $N | $N |
| Time to ship | N weeks | N weeks | N weeks |

**Cost estimate:** ~$N/month (compute $N + storage $N + network $N + managed $N)
**Riskiest assumption:** [what could fail]
**Spike candidate:** [can we validate in a day?]

### Option B: [Name]
[same structure]

### Option C: [Name]
[same structure]

## Selected Architecture

**Selected:** Option [X]
**Rationale:** [why this was chosen — reference the trade-off matrix]

### Component Breakdown

#### Component 1: [Name]
- **Responsibility:** [one sentence]
- **Inputs:** [API contracts, events, files]
- **Outputs:** [responses, events, data]
- **Dependencies (upstream):** [what it depends on]
- **Dependencies (downstream):** [what depends on it]
- **Technology:** [concrete stack and version]
- **Scaling:** [horizontal/vertical/sharding/partitioning]
- **State:** [stateless/stateful — what state]

#### Component 2: [Name]
[same structure]

### Interface Contracts

#### [Component 1] → [Component 2]
- **API:** [endpoints, request/response, error codes, auth]
- **Events:** [event names, payload schema, ordering]
- **Data:** [shared models, schema, ownership]

### Data Flow

**Write path:** [how a write enters, flows, persists]
**Read path:** [how a read is served — direct/cache/materialized]
**Cache strategy:** [what, where, invalidation]
**Async paths:** [queues, sync vs async processing]
**Data stores:** [which DB for which data, replication]

### Failure Mode Analysis

| Component | Failure | Blast Radius | Recovery | Silent/Noisy |
|---|---|---|---|---|
| [Component] | [what fails] | [what breaks] | [how to recover] | [alert or silent] |

**SPOFs:** [list]
**Cascading failure risks:** [list]
**Data loss scenarios:** [list]
**Silent failure paths:** [list]

### Deployment Topology
- **Environments:** [dev, staging, prod — differences]
- **Scaling:** [auto-scaling, instance counts, triggers]
- **Regions:** [single/multi-region, active-active/passive]
- **Network:** [VPC, subnets, security groups, CDN, DNS]
- **CI/CD:** [blue-green/canary/rolling]
- **Observability:** [metrics, logging, tracing, alerting stack]

### Migration Path (if applicable)
- **Strategy:** [strangler fig / big bang / feature flag / parallel run]
- **Steps:** [ordered migration steps]
- **Data migration:** [how to migrate data]
- **Rollback plan:** [how to revert if migration fails]

## Architecture Decision Records

### ADR-001: [Decision title]
**Context:** [why this decision is needed]
**Options considered:** [alternatives evaluated]
**Decision:** [what was decided]
**Rationale:** [why this over the others]
**Consequences:** [enables X, precludes Y, watch for Z]

### ADR-002: [Decision title]
[same structure]

## Open Questions

- [ ] OQ-1: [deferred decision — what's unknown, what's needed to decide]
- [ ] OQ-2: [deferred decision]

## Implementation Plan

### Linear Project: [Architecture Name]

### Phase 1: [Foundation — e.g., Data store + Auth]
- [ ] Ticket: [Component] — [responsibility] (priority: urgent)
- [ ] Ticket: [Spike] — [risk to validate] (priority: urgent)

### Phase 2: [Core domain]
- [ ] Ticket: [Component] — [responsibility] (priority: high)

### Phase 3: [Integration layer]
- [ ] Ticket: [Component] — [responsibility] (priority: medium)

---
*Spec by architecture-spec skill — [timestamp]*
