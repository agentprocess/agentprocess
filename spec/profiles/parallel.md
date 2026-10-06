---
title: "Profile: parallel"
description: "Fork and join: several steps ready at once, continuing when all have finished."
---

Version `parallel-1`, draft, 6 October 2026. Extends the [specification](../specification.md). Written against the 23 questions an [independent review](../../research/review-round-2.md) raised for the major-incident process, and revised after the [third review](../../research/review-round-3.md); Appendix A maps each question to the sentence that answers it.

A server that implements this profile lists `parallel-1` in `describe.profiles`. A process uses it by having a step with the `parallel:` key. No other declaration exists. A server without the profile refuses such a process at publication.

## 1. The step

```yaml
steps:
  - id: activate
    agent: Open the incident bridge and page the three leads. Record the bridge reference.
    output: { bridge: string }

  - id: respond
    parallel: [operations, security, communications]
    next: close_review

  - id: operations
    person: operations-lead
    task: Restore service. Attach the recovery evidence.
    output: { recoverySummary: string }
    evidence: [link]
    due: 1h
    next: join

  - id: security
    person: security-lead
    task: Contain the security impact and record remaining exposure.
    output: { containmentSummary: string }
    evidence: [link]
    due: 1h
    next: join

  - id: communications
    person: communications-lead
    task: Publish the customer notice and keep it updated.
    output: { communicationsSummary: string }
    evidence: [link]
    due: 1h
    next: join

  - id: close_review
    person: inputs.commander
    approve: Confirm all three leads finished with evidence.
    on_reject: respond

  - id: done
    finish: stabilized
```

`parallel:` is a kind key like `agent:` or `wait:`. Its value is a list of two or more step ids, the **branch entries**. The step takes `next` and nothing else. It does no work of its own.

## 2. Branches

A **branch** is its entry step plus every step reachable from the entry following `next` and `on_timeout` edges, stopping at `join`. `join` is a reserved word for `next` and `on_timeout` inside a branch; it means "this branch is finished". A process that uses this profile cannot have a step with id `join`.

Publication rules, in addition to the core's:

- Every branch has at least one step. Every step in a branch names `next` or `join` explicitly; the following-step default does not apply inside a branch. Every step in a branch reaches `join`.
- No step is in two branches, and no step outside the branches routes into one. A branch step never routes to a step outside its branch, to a `finish`, or to the parallel step. `join` is allowed only inside a branch.
- `on_reject` inside a branch targets a step in the same branch.
- `on_reject` on a step after the parallel step may target the parallel step, which runs every branch again, or any step before it. It may not target a step inside a branch.
- `join` edges and `parallel:` edges are not cycles for the core's acyclicity rule. Everything else is.
- A branch cannot contain a `parallel:` step. Nesting is not in version 1.
- Reachability: a branch entry is reachable through its parallel step.

## 3. Running

**Fork.** When the parallel step becomes ready, the server marks it `waiting` and makes every branch entry `ready` in the same write. Readiness is simultaneous; whether workers act at once is up to them. Each branch step then has its own claim, work item, `due` clock from this moment, escalation, `on_reject`, `wait` and events, as any step does. What the profile changes is listed in §4.

**Join.** When the last branch reaches `join`, the parallel step becomes `done` in the same write and the server makes its `next` ready. All branches must join; there is no threshold in version 1. A branch that reaches `join` early simply waits for the others; its outputs are already in run data.

**Run data.** Branch steps record their outputs under their own ids, as any step does. The parallel step records nothing in run data; its progress is its state in `steps[]`.

**Run state.** The core rule applies unchanged: `active` while any step is `ready` or `claimed`, `waiting` when every unfinished step waits on a person, timer, event or escalation.

**Ending the run from a branch.** A `failed` decision, an approval rejected with no `on_reject`, or a `wait_until` on a missing value ends the run as the core says, and the core's terminal cleanup then cancels every unfinished step in every branch and stops their tokens. A branch cannot end the run with a finish, because branches cannot contain one.

**Rework.** `on_reject` inside a branch moves that branch's later completions to history and touches no sibling. `on_reject` from after the join to the parallel step applies the core rule to the whole region: every branch step's current completion moves to history and its state becomes `null`, the rejecting approval keeps its note current, the parallel step returns to `waiting`, and every branch entry is made ready again in the same write with a fresh `due`, so the returned view shows the entries `ready` or `waiting` and the other branch steps `null`. There is no retry of one failed branch in version 1: a failed branch has ended the run.

**Snapshots.** The core takes a snapshot at read time and refuses a decision when run data changed since. Inside a fork, every sibling completion changes run data, so a decision prepared before one is refused as `stale` and costs one more read; while siblings keep completing it can be refused more than once. The person sees what the sibling did before deciding, which is the point of the check.

## 4. Run view

The parallel step appears in `steps[]` with state `waiting` from fork to join and `done` after, with `kind: "parallel"`. Branch steps appear as any other. `get_work` lists every ready agent branch step and every open branch work item. What this profile changes against the core: a new kind key; `join` as a reserved `next` value inside branches; several steps ready and several work items open at once; `parallel:` and `join` edges exempt from the acyclicity rule; and rework from after the join rewinding a whole region. Nothing else in the tool contracts changes.

## Appendix A. The review's 23 questions

| # | Question | Answer |
|---|---|---|
| 1 | What is the value of the key? | A list of branch entry step ids (§1). |
| 2 | Which step kinds may carry it? | None; `parallel:` is its own kind (§1). |
| 3 | When does the fork happen? | When the parallel step becomes ready (§3). The step records nothing in run data, so there is no timestamp to reconcile on rework. |
| 4 | Does the step itself do work? | No (§1). |
| 5 | Are branches activated atomically? | Yes, in one write (§3). |
| 6 | Simultaneous readiness or execution? | Readiness (§3). |
| 7 | How is the join identified? | `next: join` on the last step of each branch; the parallel step's own `next` is the continuation (§2, §3). |
| 8 | Does the step participate in the join? | It is the join (§3). |
| 9 | Threshold? | All branches, version 1 (§3). |
| 10 | Can a branch reach a finish early? | No; refused at publication (§2). |
| 11 | Reachability through branches? | Entries are reachable through the parallel step (§2). |
| 12 | Are these edges cycles? | `join` and `parallel:` edges are exempt; nothing else is (§2). |
| 13 | Two branches sharing a step? | Refused (§2). |
| 14 | Failure, rejection, escalation, timeout in a branch? | Per step as in core; only run-ending events affect siblings (§3). |
| 15 | Sibling cancellation on run end? | Core terminal cleanup, at once (§3). |
| 16 | Retry one failed branch? | Not in version 1 (§3). |
| 17 | Which sibling results become history on rework? | Inside a branch, only that branch; from after the join, all (§3). |
| 18 | Do sibling completions stale open snapshots? | Yes, each one; the person reads again, possibly more than once (§3). |
| 19 | How does a person get a fresh snapshot? | Every `get_run` returns a current one (core §3.4). |
| 20 | Run state with mixed agent and human branches? | Core rule unchanged (§3). |
| 21 | Nesting? | Not in version 1 (§2). |
| 22 | How is the profile declared? | By using the key; detected at publication (preamble). |
| 23 | How is the semantics version identified? | `parallel-1` in `describe.profiles` (preamble). |
