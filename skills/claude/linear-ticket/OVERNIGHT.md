# Overnight mode (`--overnight`)

Read this only when the invocation carried `--overnight` (or the user said this is
an overnight / unattended run). It replaces Phases 1–7 of `SKILL.md` at the
*orchestrator* level. Each spawned agent still runs `SKILL.md` Phases 2–7 for its
own ticket.

The premise: **the user approves once, at the keyboard, then is gone until morning.**
Everything after that gate has to be decided, not asked.

## What the flag changes

| `SKILL.md` says | Overnight |
|---|---|
| One ticket | Every ticket named, one agent each |
| `ExitPlanMode` per ticket | **One** `ExitPlanMode` at launch, covering the whole run sheet |
| "Confirm once before push/PR" | Pre-authorised at that same gate — push, PR, and the two Linear moves, for every ticket on the sheet |
| Ask the user when acceptance criteria are absent | Decide, record the assumption, report it in the morning |
| Stop and tell the user when blocked | Park the ticket, comment on it, keep the other tickets running |
| — | Never move a ticket to Done, even with `--merge`. Never merge or touch the default/integration branch directly, unless `--merge` (below) |

Pre-authorisation covers exactly what is on the approved run sheet. A ticket that
turns out to need a credential, a paid API call, a destructive migration, or a
product decision is **out of scope of that approval** — park it, do not improvise
consent from "the user said go".

## Phase O1 — Triage while the user is still awake

This is the only phase with a human in it, so it has to be worth their last ten
minutes. Do all of it before the run sheet.

1. **Resolve every ticket.** A list of identifiers, or a filter the user named
   (`search_issues` by project / label / cycle / status). Echo back the resolved set
   with titles — a filter that silently matched 23 tickets is the user's call, not
   yours.
2. **Read each one fully** — `SKILL.md` Phase 2, per ticket. This is cheap read-only
   work and it is what makes the dependency graph real instead of guessed.
3. **Check the integration branch right now**, not from the ticket text:
   ```bash
   git fetch origin <integration branch>
   git grep -n '<symbol the ticket says is missing>' origin/<integration branch> -- src tests
   git log --oneline --since=<ticket createdAt> origin/<integration branch>
   ```
   Tickets go stale in hours when several sessions merge continuously. A ticket
   already half-done on the integration branch gets re-scoped or dropped **here**,
   not seven minutes into an agent's baseline run.
4. **Find the toolchain and the verification commands once** (`SKILL.md` Phase 3),
   with absolute paths. Every agent gets handed the result; none of them should
   rediscover that the build tool is not on `PATH` at 3am.
5. **Classify each ticket** — `buildable` or `needs-a-human` (see the hard stops at
   the bottom). Needs-a-human tickets come off the sheet now, with a one-line reason.

Then present the **run sheet** and call `ExitPlanMode` once:

- the waves, in order, and which tickets run in parallel within each;
- for each ticket: branch name (Linear's `gitBranchName`), base branch, worktree path;
- the PR topology — which PRs cascade onto which, which go straight to the integration branch;
- the verification command set every agent will run;
- the per-ticket budget and what "parked" will look like in the morning;
- the tickets dropped as needs-a-human;
- whether `--merge` is on.

Do not create a single worktree before that approval.

## Phase O2 — The dependency graph

An edge exists when any of these hold. Check all four; only the first is visible in
Linear's relation graph.

1. **`blocks` / `blocked by` relations** from `get_issue(includeRelations: true)`.
2. **Parent/sub-issue phase order** — an umbrella ticket's sub-issues named
   "Phase 1/2/3", or descriptions that reference each other's output.
3. **File overlap.** Two tickets whose planned edits touch the same file conflict
   whether or not Linear knows it. This is the edge that gets missed. Derive it from
   the Phase 3 grounding you did in O1: list the files each ticket will touch, and
   sequence any pair that shares one. Schema/version/registry files — anything with a
   monotonic number or a central list — count double; two tickets bumping the same
   version constant always collide.
4. **An order the user stated.** It wins over your inference.

Then:

- **No edges → parallel.** Branch off `origin/<integration branch>`, PR straight to it,
  merge in any order. Say "independent, mergeable in any order" in the PR body.
- **Edges → one sequential chain, cascading branches.** Ticket B that depends on A
  branches off **A's branch**, not the integration branch, and opens its PR with
  `--base <A's branch>`. B starts as soon as A's PR is open — not when it is reviewed.
- A chain is one agent handling its tickets in order, **not** a new agent per link.
  One agent already holds the context for A when it starts B, and re-briefing a
  fresh agent onto a half-built stack is where stacks go wrong.
- Cap the fan-out. Past ~4–5 concurrent agents the shared machine (CPU, one
  toolchain, one integration branch) is the bottleneck and everything gets slower
  and harder to attribute. Queue the rest behind the first wave.

## Phase O3 — Dispatch

**Isolation is the whole game.** Parallel agents inherit the same paths by default
and overwrite each other silently — no error, just wrong results.

| Shared by default | Give each agent |
|---|---|
| Worktree dir (named from the branch) | Its own — one ticket, one branch, one directory. Two agents on one ticket is the single worst outcome. |
| Scratchpad dir | `scratchpad/<TICKET-ID>/` — otherwise one agent's test log lands in another's file |
| The project's user-state / cache dir (keyed by project name, so every worktree shares it) | A per-agent isolated one — the project's own override mechanism, or an isolated `HOME` for the test command only. **Never give git or `gh` the isolated `HOME`** — they lose the user's identity and auth. |
| Any single-instance local resource the suite needs | If the runtime genuinely allows one process at a time, serialise on an atomic `mkdir` lock acquired **per run, not per batch**, released in an `EXIT` trap. Do **not** make agents wait on `pgrep` being empty — with other sessions running it never clears and the agent stalls for the rest of the night. |

The brief for each agent must stand alone — it cannot ask you anything:

- ticket identifier + the acceptance criteria you extracted, and the fact that it
  should still read the ticket itself (`SKILL.md` Phases 2–7);
- its branch, its **base** branch, its worktree path, its scratchpad subdir, its
  isolated state dir;
- the exact verification commands and the absolute toolchain path from O1;
- **"Overnight run: nobody will answer you. Decide, record the assumption in your
  final report, and keep going. Push, open the PR, and move the ticket — all
  pre-authorised. Do not merge anything."**
- **"Stay on your own branch and worktree. Do not touch another agent's directory,
  branch, or the integration branch."**
- **"Block on long runs inside your turn"** — background the run (`nohup … & echo $! >
  run.pid`) and `Monitor` until it exits, rather than ending your turn to wait.
  An agent that ends its turn mid-work is the thing that makes the rest of this
  protocol hard.
- its budget, and what to do on exhaustion (Phase O4).

## Phase O4 — Orchestrating while the user sleeps

**Never end your turn while agents are running.** A completed child whose parent's
turn has ended is orphaned work. You own collection.

**A "finished" notification can be a pause.** It fires every time an agent stops with
no live background children — including an agent that backgrounded its own test run
and will be woken when it finishes. Before concluding an agent is dead:

- wait for a second notification on the same task id;
- check whether its branch or worktree is still moving — new commits, fresh scratch
  files, a live process whose cwd is inside its worktree (`lsof -a -p <pid> -d cwd -Fn`);
- a process check that finds *nothing* while other agents are known to be testing is
  a broken check, not reassurance.

**Never run two agents on one ticket.** If a replacement is genuinely needed:
`TaskStop` the original, confirm it stopped, *then* brief the replacement. Once two
agents share a worktree, nothing in git can tell you which one is acting — the
commits, merges and Linear comments all look identical — and stopping the wrong one
leaves nobody on the ticket with half-applied work.

**Between waves**, before dispatching anything that branches off what just landed:

- re-fetch the integration branch and re-check the next ticket's premises against it;
- for every open stacked branch, `git merge-tree --write-tree --name-only
  origin/<integration> origin/<branch>` (exit 1 = conflict). GitHub rates a stacked
  PR against its *predecessor*, so it will say MERGEABLE while conflicting with the
  integration branch. Resolve bottom-up, and note it in the morning report.

**Budgets and parking.** Give each ticket a wall-clock budget and an attempt cap
(e.g. 3 red verification runs in a row). On exhaustion, the agent stops cleanly:

- push the branch and open a **draft** PR whose body is the actual failure output —
  something to look at beats nothing;
- comment the blocker on the ticket;
- move the ticket **back to the status it started in**, so morning triage is honest.
  A ticket stranded in In Progress with nobody on it is worse than one in Todo.
- its dependents do not start. Do not stack onto a broken base; report them as
  "not started — blocked by <ticket>".

One bad ticket must not eat the night. Park it and keep the others moving.

## Phase O5 — The morning report

One message, written to be read cold with no memory of the night. Per ticket:
status, PR URL, what verification actually printed, assumptions made, anything
deliberately left out. Then:

- **the recommended merge order**, stack by stack, and any `merge-tree` conflict found;
- parked tickets, with the blocker and the question the user needs to answer;
- anything that hit a hard stop and why;
- with `--merge`: every **owed** check per ticket (what to run, where, and what result
  counts as passing), plus the leftovers to clean up.

Post the same summary as a comment on the umbrella ticket if there is one. Send the
report file with `SendUserFile` so it is on their phone.

## `--merge` (opt-in, off by default)

`--overnight` alone leaves every PR open; the cascade is the deliverable. With
`--merge` the orchestrator may also land a stack **whose every PR is green** — green
meaning the local gate below, with red CI checks resolved per "A red CI check may be a billing refusal" below —
and only that way:

- merge the integration branch *into* the stack bottom-up first, resolving at the
  branch that introduced the clash, so each PR diff stays clean;
- run the full verification suite on the resulting top tree — **stop on a failed
  merge**; a gate that runs on regardless just tests a tree full of conflict markers;
- re-read the integration branch tip in the same command as the merge — it moves
  between your check and your push;
- never force-push; never merge a red or draft PR.

**Then land the PRs top-down — the opposite direction from the fold above.** Merge
the top PR into its parent branch first, then that parent into *its* parent, and so
on, so the bottom PR's merge into the integration branch carries the whole stack.
Top-down works because every PR's base branch still exists when it merges.

Bottom-up does not. Once the bottom PR has landed, the next PR up still targets the
bottom branch, which has already merged. Merging it puts its commits on that dead
branch, and they never reach the integration branch. GitHub only retargets a stacked
PR when its base branch is **deleted**, so leaving branches in place (which is
correct) means nothing retargets for you. This happened on the first real run.
If you find it has happened, stop merging. Confirm what each remaining branch
carries against the integration branch, then `gh pr edit <n> --base <integration>`
the next open PR and continue. A stacked branch contains its predecessors' commits,
so the stranded commits come across with it.

After the last merge, **assert the landed tree is the gated tree**:
`git diff --quiet <gated top-of-stack commit> origin/<integration>` must exit 0,
provided nothing else landed on the integration branch meanwhile. If it doesn't, what
shipped is not what you tested. Say so in the report.

If any of that fails, stop merging and leave everything for the morning. A failed
overnight merge is not something to retry three times.

### Checks the ticket asks for that the night cannot run

A ticket's own Verification section often asks for things the local gate cannot do:
live model calls, a different machine, a manual playthrough, a measurement over many
runs. Sort each one before merging:

- **It measures an effect** (does the change do enough? distinct names, fallback
  rate, timings). It does not block the merge. Merge on the local gate and list the
  check as **owed**.
- **It guards correctness or safety** (something must never happen: zero network
  calls, no data loss, a migration that round-trips). It blocks the merge **unless a
  test in the gated suite already pins the same guarantee**. Name that test in the
  report. If no such test exists, leave the PR open and say why.

When you can't tell which kind a check is, treat it as a guard.

### After merging: ticket status and leftovers

**Leave every merged ticket In Review. Never move it to Done.** Merging here is not
review: nobody has read the code, and owed checks may still be pending. Comment on
each ticket with the merge commit and its owed checks. The user closes it.

**List the leftovers, don't delete them.** That means remote branches, the local gate
branch, worktrees, and per-agent state dirs. Deleting is outward-facing, and it is not
on the run sheet. A session also cannot remove its own worktree. Never delete a base
branch while PRs above it are still open, because GitHub then retargets them in ways
you did not plan.

### A red CI check may be a billing refusal, not a result

GitHub Actions on a private repo is billed, and when payment fails **GitHub refuses
to start the jobs and marks them red**. A gate that never ran and a gate that caught
a real problem look identical in `gh pr checks` — both just say `fail`. Treating the
first as a blocker stalls the whole night; treating the second as billing merges a
real defect. So diagnose, per failing check, per PR — never infer it from "billing was
broken yesterday", because it may have been fixed overnight and the red is real again.

A job that never started has **all** of these signatures:

```bash
gh api repos/{owner}/{repo}/actions/jobs/<job-id>          # "steps": 0, empty runner_name,
                                                           # started_at ≈ completed_at (1-2s)
gh run view <run-id> --log-failed                          # "log not found"
gh api repos/{owner}/{repo}/check-runs/<job-id>/annotations # the only place the reason exists
```

The annotation is the proof — it reads *"The job was not started because recent
account payments have failed or your spending limit needs to be increased."*

- **All signatures present → the check is not evidence of anything.** Do not count it
  as passed, and do not let it block the merge. The local gate becomes the only
  evidence. Quote the annotation in the morning report.
- **Any step ran, or a log exists → the check really failed.** Stop. Do not merge.
  Report it with the actual output. A slow, flaky, or queued job is also not a
  billing refusal — wait for it or leave the PR open.
- Billing is the user's account to fix. Never touch billing or spending settings,
  and never disable, skip, or edit a workflow to make a red check go away.

**What "green locally" has to mean before merging past an unrun CI.** The bar goes
up, not down, because nothing else is checking:

- run **every step CI would have run**, not just the test suite — the compile /
  typecheck / lint steps too;
- run them on the **gated merge tree** (the top of the stack after folding the
  integration branch in), not on each branch separately;
- apply `SKILL.md` Phase 6 rigour — compare the test total against the baseline and
  grep for parse/collection errors, since a suite that silently drops a file still
  exits green;
- if a **secret-scanning** gate is among the unrun checks, nothing scanned the diff.
  Substitute a local scan of `git log -p <base>..<branch>` for credential patterns,
  or a pinned local scanner **if a verified copy is already on the machine** —
  downloading a binary overnight is not covered by the launch approval.

Record all of it in the merge comment and the morning report: which checks were red,
the annotation proving each never ran, and exactly which local commands stood in for
them. **Never write "CI green" about a check that never started.**

## Never overnight — drop these at O1

- Anything needing a credential, a secret, a paid API call, or a service account.
- Anything destructive or hard to reverse: data migrations, deletions, force-pushes,
  releases, infra changes, anything touching production.
- A ticket whose acceptance criteria are contradictory or absent *and* whose answer
  is a product decision — guessing burns a night and produces a PR that gets closed.
- A ticket whose blocking relation is still open.
- A ticket already `In Progress` — another session may be mid-run on it; a clean
  worktree is not proof it is free.

Name these on the run sheet so the user can answer the question before bed and add
the ticket back in.

## Failure modes this mode exists to prevent

- Ending the turn with agents still running, orphaning a night of work.
- Two agents in one worktree — from a "finished" notification that was a pause.
- Parallel agents clobbering each other's logs or shared state dir, producing test
  numbers nobody can trust.
- Stacking onto a branch that never went green.
- A stack that is MERGEABLE against each predecessor and conflicts with the
  integration branch at the bottom.
- Merging a stack bottom-up, so a child PR lands on a parent branch that has already
  merged, and its commits never reach the integration branch.
- Merging a ticket whose safety guarantee is only checked by a manual step nobody ran,
  or merging without listing the checks still owed.
- One stuck ticket consuming the whole night while five buildable ones wait.
- Tickets left stranded In Progress with nobody on them.
- A morning report that says "done" about a check that was never run.
- Reading a billing-refused CI job as a real failure and stalling a merge on it —
  or the reverse, waving a genuine failure through as "that's just the billing thing".
