# Independent round-three review of agentprocess

This review uses only `agentprocess-core-for-review.md`, `agentprocess-tools-for-review.md`, and `agentprocess-parallel-for-review.md`. They are treated together as the specification. No implementation, previous review, or previous review round was consulted. These are hypothetical contract traces, not test executions.

## Reading conventions and principal findings

References below use **C** = `agentprocess-core-for-review.md`, **T** = `agentprocess-tools-for-review.md`, and **P** = `agentprocess-parallel-for-review.md`, followed by source line numbers. Line references identify the supplied versions of those files.

All run IDs, work-item IDs, tokens, snapshot strings, document references, timestamps, and request IDs in the traces are **illustrative values**. An actual client copies opaque values returned by its server. In particular, `s0`, `s1`, etc. stand for different opaque snapshot hashes, not an algorithm or promised hash encoding. Evidence references in examples are illustrative document numbers unless explicitly identified as file IDs. No external bridge, notice, approval, receipt upload, or signature was actually created.

Every JSON **call** below contains all the arguments sent in that call. Every JSON **result projection** contains only the exact named fields the actor reads or the review relies on; omitted fields are not claimed absent. This matters because the contracts promise a whole run view on many writes. The two nested `data` keys in a run-view result are intentional: the outer one is the result envelope, and the inner one is run data. Error `message` is required but its wording is not specified; error projections therefore show `code`, and show `issues` only where the contract promises those particular issues. No sample error wording is represented as guaranteed.

The significant findings are:

1. **The all-branches barrier is specified.** Separate branch outputs, simultaneous readiness, and postponing close review until every branch joins are explicit. A normal incident execution can be traced without inventing an orchestration API.
2. **The human surface is data, not an executable action affordance.** It supplies assignments, instructions, schemas, and snapshots, but no equivalent of `get_work.items[].claim`. A client that already knows the tool contracts can operate it. Three people who know neither the documents nor a contract-aware UI are not supplied the complete recipe by the work-item result alone.
3. **Parallel human decisions intentionally conflict with sibling completions.** Retrying is specified; being refused only “once” is not guaranteed. The profile makes unrelated completed work invalidate a person's pending decision, and cannot guarantee eventual success under continuing changes.
4. **Terminal cleanup has stronger semantics than terminal error reporting.** Outstanding work disappears and tokens stop working. The precise reply to the next submission or decision after cancellation/failure, including error precedence, is not fully specified.
5. **Rework does not have a complete observable record contract.** Branch current values move to history, but the parallel step's own timestamp/history, the rejected approval's current note versus history, and states of downstream steps awaiting re-entry are not consistently defined.
6. **One-hour due values are soft readiness clocks.** They start after activation is accepted and the fork opens, not necessarily at incident declaration, the commander's order, notification receipt, or the start of actual response work.

## Task 1 — Major incident

### A complete PROCESS.md

The containing folder is `major-incident`, matching the declared name as required by C:63. At import the four lead/commander identities must be resolvable: the three role names map to groups; the commander is an input of type `person`. Publication and role mapping occur outside the requested `start_run` trace; no publication tool is invented. The server must advertise `parallel-1` (P:5). The input value `commander@example.org` assumes a server whose `describe.person.format` is `email`; that spelling is not universally portable (C:103).

```markdown
---
name: major-incident
description: Open an incident bridge, coordinate three response leads, and obtain commander closure after all three finish.
inputs:
  incidentId: string
  commander: person
steps:
  - id: activate
    agent: |
      Open the incident bridge for inputs.incidentId and page the operations,
      security, and communications leads. Record the bridge reference.
    output:
      bridge: string
    evidence: [link]
    next: respond

  - id: respond
    parallel: [operations, security, communications]
    next: close_review

  - id: operations
    person: operations-lead
    task: |
      Use steps.activate.output.bridge. Restore service, record what was
      recovered and what remains at risk, and attach recovery evidence.
      On rework read the retained history and the commander's rejection note.
    output:
      recoverySummary: string
    evidence: [link]
    due: 1h
    next: join

  - id: security
    person: security-lead
    task: |
      Use steps.activate.output.bridge. Contain the security impact, record
      remaining exposure, and attach containment evidence.
      On rework read the retained history and the commander's rejection note.
    output:
      containmentSummary: string
    evidence: [link]
    due: 1h
    next: join

  - id: communications
    person: communications-lead
    task: |
      Use steps.activate.output.bridge. Publish the customer notice, record
      its status, and attach the published notice as evidence.
      On rework read the retained history and the commander's rejection note.
    output:
      communicationsSummary: string
    evidence: [link]
    due: 1h
    next: join

  - id: close_review
    person: inputs.commander
    approve: |
      Inspect all three current branch summaries and evidence references.
      Approve closure only when each lead's work is complete and the evidence
      supports closure. Otherwise reject with concrete rework instructions.
    on_reject: respond
    next: done

  - id: done
    finish: stabilized
---

# Major incident response

The agent opens the bridge and pages the leads. Operations, security, and
communications receive separate work items and work concurrently. Their
one-hour due times start when their steps become ready; overdue work remains
open. Each lead supplies the required summary and evidence reference.

The commander receives close review only after all three branches finish.
Rejecting close review reopens every branch. Earlier branch completions are
retained in history and must be considered during rework. Evidence links are
references; participants must inspect the underlying records themselves.
```

This enforces completion and evidence-kind presence before close review, not successful recovery or the quality of the underlying evidence. `link` is explicitly verified as “Nothing. Recorded as stated” (C:231); business correctness is a non-goal (C:13). The word “deadlines” therefore means the specified overdue/notification behavior, not mandatory completion within an hour.

### 1a. Happy path, from start_run to end

Assume successful authentication, published version 1, mapped roles, no lease expiry, and no intervening changes beyond those shown. Each actor has the run ID through organizational handoff; human run discovery is not supplied by `get_run` itself. Work execution is concurrent even though these acceptance calls are shown in the server's serialization order. The leads can all begin work at 09:05, then read current state immediately before finishing.

**1. Agent starts the run.**

```json
start_run({"process":"major-incident","version":1,"inputs":{"incidentId":"INC-42","commander":"commander@example.org"},"mode":"live","requestId":"ia-start"})
```

Result projection:

```json
{"ok":true,"data":{"run":{"id":"r_inc","process":"major-incident","version":1,"mode":"live","state":"active"},"steps":[{"id":"activate","kind":"agent","state":"ready"}],"workItems":[]}}
```

The actor reads `data.run.id`, `data.run.mode`, and the `activate` state. The `steps` projection is filtered to the relevant entry; the actual array includes other steps in their unreached states (T:25–39).

**2. Agent obtains the claim call.**

```json
get_work({"process":"major-incident"})
```

Result projection for this run:

```json
{"ok":true,"data":{"items":[{"runId":"r_inc","stepId":"activate","mode":"live","claim":{"tool":"claim","arguments":{"runId":"r_inc","stepId":"activate","requestId":"<new UUID>"}}}],"nextCursor":null}}
```

The literal placeholder means the client generates a new request ID, following T:86. It is not sent as the request ID below. Filtering only by process can return other runs; the actor matches `runId`, and would page using `nextCursor` when necessary. This trace assumes this is the only matching item.

**3. Agent claims activation.**

```json
claim({"runId":"r_inc","stepId":"activate","requestId":"ia-claim"})
```

Result projection:

```json
{"ok":true,"data":{"token":"token-activate","expiresAt":"2026-10-06T09:10:00Z","mode":"live","step":{"id":"activate","instructions":"Open the incident bridge for inputs.incidentId and page the operations,\nsecurity, and communications leads. Record the bridge reference.\n","output":{"bridge":"string"},"evidence":["link"]},"data":{"inputs":{"incidentId":"INC-42","commander":"commander@example.org"}}}}
```

The agent also reads `data.body`. Its value is exactly the body in the PROCESS.md above. The instruction string follows the YAML literal block. The agent reads its token, expiry, mode, instructions, output schema, and evidence requirement. It uses its own external tools to open and page; those operations have no tool contracts in these three documents. Their actual success is assumed, not proved by `submit`.

**4. Agent submits the bridge result at illustrative 09:05.**

```json
submit({"token":"token-activate","output":{"bridge":"BRIDGE-42"},"evidence":[{"kind":"link","ref":"BRIDGE-42"}],"requestId":"ia-submit"})
```

No `next` or `reason` is sent: the declared `next` is a scalar, and sending either is refused (C:130).

Result projection after the fork and person-step activation:

```json
{"ok":true,"data":{"run":{"id":"r_inc","state":"waiting"},"data":{"steps":{"activate":{"output":{"bridge":"BRIDGE-42"},"evidence":[{"kind":"link","ref":"BRIDGE-42"}]}}},"steps":[{"id":"respond","kind":"parallel","state":"waiting"},{"id":"operations","kind":"task","state":"waiting","due":"2026-10-06T10:05:00Z","overdue":false},{"id":"security","kind":"task","state":"waiting","due":"2026-10-06T10:05:00Z","overdue":false},{"id":"communications","kind":"task","state":"waiting","due":"2026-10-06T10:05:00Z","overdue":false},{"id":"close_review","kind":"approve","state":null}],"workItems":[{"id":"w_ops","stepId":"operations"},{"id":"w_sec","stepId":"security"},{"id":"w_comms","stepId":"communications"}]}}
```

P:70 expressly makes all three entries `ready` “in the same write.” C:153–155 creates the human work items, and P:86 includes all open branch work items. The documents do **not expressly say that all automatic ready-to-waiting transitions and work-item creation are drained before the initiating `submit` returns**. Thus this projection is the natural quiescent result, but its immediate timing is conditional. A conforming view might first show the fork's `ready` entries; subsequent `get_run({"runId":"r_inc"})` calls can observe the work items once created. There is no contractual polling interval or time bound. The simultaneous-readiness guarantee itself does not depend on this presentation question.

**First unspecified point in this trace:** the response boundary in step 4: whether the returned run view already contains every work item, or exposes a transient ready state while automatic transitions proceed. All following calls are conditional on those work items existing. This is a response-timing gap, not an undefined fork or join rule. The same timing qualification applies to later automatic join/finish transitions; no stronger assumption is silently introduced.

**5. Operations reads the run when its work is ready to record.**

```json
get_run({"runId":"r_inc"})
```

It reads `data.run.id`, `data.body`, the whole `data.data`, and this work-item projection:

```json
{"ok":true,"data":{"run":{"id":"r_inc"},"workItems":[{"id":"w_ops","stepId":"operations","kind":"task","text":"Use steps.activate.output.bridge. Restore service, record what was\nrecovered and what remains at risk, and attach recovery evidence.\nOn rework read the retained history and the commander's rejection note.\n","output":{"recoverySummary":"string"},"evidence":["link"],"next":null,"assignedTo":{"group":"g_ops"},"snapshot":"s0","due":"2026-10-06T10:05:00Z","escalation":null}]}}
```

The work item does not carry an allowed `next` list because `join` is a scalar. `next:null` here denotes the absence of a client route choice, not absence of the server's declared join route (T:40).

**6. Operations records completion.**

```json
decide({"runId":"r_inc","workItemId":"w_ops","snapshot":"s0","decision":"completed","output":{"recoverySummary":"Service restored; recovery checks passed."},"evidence":[{"kind":"link","ref":"RECOVERY-42"}],"requestId":"io-done"})
```

It reads:

```json
{"ok":true,"data":{"run":{"state":"waiting"},"data":{"steps":{"operations":{"output":{"recoverySummary":"Service restored; recovery checks passed."},"evidence":[{"kind":"link","ref":"RECOVERY-42"}],"by":"u_ops","at":"2026-10-06T09:25:00Z"}}},"steps":[{"id":"operations","state":"done"},{"id":"respond","state":"waiting"}]}}
```

The operations item is absent from `workItems`; security and communications remain. Their returned snapshots reflect the current read, not necessarily the earlier `s0`.

**7. Security reads after operations' completion.**

```json
get_run({"runId":"r_inc"})
```

Security reads the whole run data, including the above `data.data.steps.operations`, and:

```json
{"ok":true,"data":{"run":{"id":"r_inc"},"workItems":[{"id":"w_sec","stepId":"security","kind":"task","output":{"containmentSummary":"string"},"evidence":["link"],"next":null,"assignedTo":{"group":"g_security"},"snapshot":"s1","due":"2026-10-06T10:05:00Z","escalation":null}]}}
```

Security also reads `workItems[].text` and `data.body`, whose exact values are the security task and process body above.

**8. Security records completion.**

```json
decide({"runId":"r_inc","workItemId":"w_sec","snapshot":"s1","decision":"completed","output":{"containmentSummary":"Compromised access revoked; no known remaining exposure."},"evidence":[{"kind":"link","ref":"CONTAINMENT-42"}],"requestId":"is-done"})
```

It reads `ok:true`, `data.steps[id=security].state:"done"`, `data.steps[id=respond].state:"waiting"`, and `data.data.steps.security.output/evidence/by/at`. The returned output and evidence exactly match the submitted values; illustrative `by` is `u_security`, and `at` is `2026-10-06T09:30:00Z`. Its work item disappears. There is still no close-review item.

**9. Communications reads after security's completion.**

```json
get_run({"runId":"r_inc"})
```

It reads the whole run data, the body, its task text, and:

```json
{"ok":true,"data":{"run":{"id":"r_inc"},"workItems":[{"id":"w_comms","stepId":"communications","kind":"task","output":{"communicationsSummary":"string"},"evidence":["link"],"next":null,"assignedTo":{"group":"g_comms"},"snapshot":"s2","due":"2026-10-06T10:05:00Z","escalation":null}]}}
```

**10. Communications completes; all branches have joined.**

```json
decide({"runId":"r_inc","workItemId":"w_comms","snapshot":"s2","decision":"completed","output":{"communicationsSummary":"Customer notice published and current."},"evidence":[{"kind":"link","ref":"NOTICE-42"}],"requestId":"ic-done"})
```

At the joined state it reads:

```json
{"ok":true,"data":{"run":{"state":"waiting"},"data":{"steps":{"communications":{"output":{"communicationsSummary":"Customer notice published and current."},"evidence":[{"kind":"link","ref":"NOTICE-42"}],"by":"u_comms","at":"2026-10-06T09:35:00Z"},"respond":{"at":"2026-10-06T09:35:00Z"}}},"steps":[{"id":"respond","state":"done"},{"id":"close_review","state":"waiting"}],"workItems":[{"id":"w_close","stepId":"close_review","kind":"approve","output":null,"evidence":null,"next":null,"snapshot":"s3"}]}}
```

P:72 specifies that the last join marks the parallel step done and makes its next step ready in the same write. P:74 promises `respond.at` and no other parallel completion fields. The example time assumes it is a completion time; that timestamp's precise semantic definition is not independently given by the profile.

**11. Commander reads current evidence.**

```json
get_run({"runId":"r_inc"})
```

The commander reads `data.run.id`, `data.body`, `data.data.steps.operations`, `.security`, `.communications`, and `workItems[id=w_close].id/kind/text/snapshot/output/evidence/next`. The schema represents `assignedTo` as an object and illustrates its group case; the exact encoding of a direct-person assignment is not specified by T:33. No invented `{person:...}` field is used here. Authorization is nonetheless stipulated by the `inputs.commander` assignment.

Assume no intervening run-data change, so `snapshot` is still `s3`.

**12. Commander approves.**

```json
decide({"runId":"r_inc","workItemId":"w_close","snapshot":"s3","decision":"approved","requestId":"im-close"})
```

At the terminal state it reads:

```json
{"ok":true,"data":{"run":{"id":"r_inc","state":"ended","outcome":"stabilized"},"data":{"steps":{"close_review":{"decision":"approved","by":"u_commander","at":"2026-10-06T09:40:00Z"}}},"workItems":[]}}
```

If the returned view precedes execution of `finish`, the commander follows with `get_run({"runId":"r_inc"})` to observe this terminal state. The exact value or absence convention for an unused approval `note` is not specified; no `note:null` guarantee is invented. The all-branches prerequisite is settled. The elapsed interval from commander's approval to automatic finish becoming observable is not bounded by the contracts.

### 1b. Security reads; operations completes; security decides

Start from the three open work items, with no completions. Security has done its work and is preparing to record it. The following is the complete decision race; independent external work is not a process-server call.

1. Security calls `get_run({"runId":"r_inc"})`. It reads `data.workItems[id=w_sec].snapshot:"b0"`, `.id:"w_sec"`, task output/evidence requirements, and run data with no current operations completion.
2. Operations calls `get_run({"runId":"r_inc"})`. It reads `data.workItems[id=w_ops].snapshot:"b0"`, `.id:"w_ops"`, and its requirements.
3. Operations sends:

```json
decide({"runId":"r_inc","workItemId":"w_ops","snapshot":"b0","decision":"completed","output":{"recoverySummary":"Service restored; recovery checks passed."},"evidence":[{"kind":"link","ref":"RECOVERY-42"}],"requestId":"ib-ops"})
```

Result projection: `{"ok":true,"data":{"steps":[{"id":"operations","state":"done"}],"data":{"steps":{"operations":{"output":{"recoverySummary":"Service restored; recovery checks passed."},"evidence":[{"kind":"link","ref":"RECOVERY-42"}]}}}}}`. The operations result may itself contain the now-current security snapshot, but security does not automatically receive that result.

4. Security sends its prepared decision with the stale snapshot:

```json
decide({"runId":"r_inc","workItemId":"w_sec","snapshot":"b0","decision":"completed","output":{"containmentSummary":"Access revoked; containment verified."},"evidence":[{"kind":"link","ref":"CONTAINMENT-42"}],"requestId":"ib-sec"})
```

Result projection: `{"ok":false,"error":{"code":"stale"}}`. The actual result also has an unspecified `message`. This decision records nothing, and the work item remains open (C:166; T:165).

5. Security calls `get_run({"runId":"r_inc"})`. It reads the operations completion from `data.data.steps.operations`, confirms that `w_sec` remains present, reconsiders its own conclusion in that context, and copies `data.workItems[id=w_sec].snapshot:"b1"`.
6. Provided nothing changes after that read, security sends:

```json
decide({"runId":"r_inc","workItemId":"w_sec","snapshot":"b1","decision":"completed","output":{"containmentSummary":"Access revoked; containment verified."},"evidence":[{"kind":"link","ref":"CONTAINMENT-42"}],"requestId":"ib-sec"})
```

Result projection: `{"ok":true,"data":{"steps":[{"id":"security","state":"done"}],"data":{"steps":{"security":{"output":{"containmentSummary":"Access revoked; containment verified."},"evidence":[{"kind":"link","ref":"CONTAINMENT-42"}]}}}}}`. `w_sec` is absent from the returned open items.

Reusing `ib-sec` with a fresh snapshot is allowed here: “A call that failed is not recorded” (C:184). Had the first attempt succeeded, changed arguments with that request ID would conflict. A fresh request ID would also be valid after a known failure.

**First unspecified point:** none for this finite sequence, conditional only on no new change between calls 5 and 6 and on already-existing work items. The generated hash bytes and error-message wording are intentionally not prescribed and are not recovery gaps. If communications completes between calls 5 and 6, call 6 returns `stale` again. Thus P:82's “refused once and reads again” is an inaccurate guarantee if read literally. There is no atomic read-and-decide tool and no bound on repeated refusal under continuing sibling changes.

### 1c. Operations fails while security is working and communications has an agent claim

**The baseline process cannot have the stated communications claim.** Communications is a human `task`. An agent calling:

```json
claim({"runId":"r_inc","stepId":"communications","requestId":"ic-illegal-claim"})
```

gets `{"ok":false,"error":{"code":"forbidden"}}` because this is a non-agent step (T:107). This is a specified rejection, not an unspecified behavior.

To examine the requested mixed human/agent failure state, use a separately published incident variant in which the communications step is exactly:

```yaml
  - id: communications
    agent: Publish the customer notice, record its status, and attach the notice reference.
    output:
      communicationsSummary: string
    evidence: [link]
    due: 1h
    next: join
```

All other steps are the incident process above. This variant intentionally changes who performs communications; it does not pretend the original human task is claimable. Assume the variant's run ID is `r_mix`, the fork is open, and the bridge agent has already completed activation.

1. Security calls `get_run({"runId":"r_mix"})`; reads its open `w_mix_sec`, `snapshot:"c0"`, task text, output/evidence requirements, bridge data, and starts or continues containment.
2. Communications agent calls `get_work({"process":"major-incident-agent-comms"})`; reads an item with `runId:"r_mix"`, `stepId:"communications"`, and `claim.arguments` for those values. It sends:

```json
claim({"runId":"r_mix","stepId":"communications","requestId":"ic-agent-claim"})
```

Result projection: `{"ok":true,"data":{"token":"token-comms","expiresAt":"2026-10-06T09:20:00Z","mode":"live","step":{"id":"communications","instructions":"Publish the customer notice, record its status, and attach the notice reference.","output":{"communicationsSummary":"string"},"evidence":["link"]}}}`. It also reads the supplied body and run data. The task is now `claimed`; the run is `active` (P:76).

3. Operations calls `get_run({"runId":"r_mix"})`; reads its still-open `w_mix_ops` and current `snapshot:"c_now"`. No equality between this hash and `c0` is assumed: the subsequent failure case does not need one.
4. Operations sends:

```json
decide({"runId":"r_mix","workItemId":"w_mix_ops","snapshot":"c_now","decision":"failed","note":"Recovery cannot proceed because the production data copy is unusable.","requestId":"ic-ops-fail"})
```

Guaranteed result projection:

```json
{"ok":true,"data":{"run":{"id":"r_mix","state":"ended","outcome":"failed"},"steps":[{"id":"respond","state":"cancelled"},{"id":"security","state":"cancelled"},{"id":"communications","state":"cancelled"}],"workItems":[]}}
```

The claim stops working; there is no close review. P:78 expressly applies terminal cleanup to every branch. Activation's earlier completion remains a historical fact; nothing rolls back bridge creation or a partially published notice (C:174).

**First unspecified point in the mixed trace:** the successful failure decision's representation of the **operations step itself and its failure note**. `failed` ends the run, but the allowed step states contain no `failed`, and neither the run-data shape nor the tools shape says whether the deciding task becomes `done`, becomes `cancelled` as unfinished, or where its failure note is recorded. C:121 describes output/evidence/next/reason for completed tasks; T:43 describes only approval decision records. The projection above deliberately excludes an invented operations record. The terminal effects on the *other*, unfinished branches are explicit.

Every participant's next interaction, conditional on the accepted failure:

| Participant | Exact next call | What is promised; what is not |
|---|---|---|
| Operations lead | `get_run({"runId":"r_mix"})` | `data.run.state:"ended"`, `outcome:"failed"`, and `workItems:[]`; no guaranteed structured location for its failure note. |
| Security lead, attempting to finish without first rereading | `decide({"runId":"r_mix","workItemId":"w_mix_sec","snapshot":"c0","decision":"completed","output":{"containmentSummary":"Containment completed."},"evidence":[{"kind":"link","ref":"CONTAINMENT-MIX"}],"requestId":"ic-sec-late"})` | It cannot complete removed/cancelled work. The exact error code and precedence are unspecified: a removed item is not an “already decided” item by an explicit rule, and `stale` is specified for a changed run-data snapshot, not for every cancelled item. No success is possible under terminal cleanup. |
| Security lead recovering | `get_run({"runId":"r_mix"})` | Sees `ended/failed`, its step `cancelled`, no work items. Its ongoing real containment work was not interrupted by a process-server push promised in these tools. |
| Communications agent, attempting to finish | `submit({"token":"token-comms","output":{"communicationsSummary":"Notice published."},"evidence":[{"kind":"link","ref":"NOTICE-MIX"}],"requestId":"ic-comms-late"})` | Rejected because the token stopped working. The generic stale-token description suggests `stale` (C:218), but T:138 enumerates lease expiry/replacement, not terminal invalidation; no precise terminal-token error rule or precedence is supplied. |
| Communications agent recovering | `get_run({"runId":"r_mix"})` | Sees its step `cancelled`, the run `ended/failed`, and no human work items. |
| Commander | `get_run({"runId":"r_mix"})` | Sees `ended/failed`, no close-review item. The run did not wait for commander authorization to terminate. |
| Bridge/activation agent | `get_run({"runId":"r_mix"})` | Sees the same terminal state, and its completed activation data. It is not assigned a new cleanup task. |

An operator reading the run sees the same terminal view. The tools promise no cancellation notification payload to any of these actors. Existing real work may continue until actors next observe server state.

### 1d. Commander rejects close_review back to respond

Start from the joined state in 1a. All three current branch completions exist and `w_close` is open.

1. Commander calls `get_run({"runId":"r_inc"})`; reads `workItems[id=w_close].snapshot:"d0"`, the approval text, and the three current outputs/evidence.
2. Commander sends:

```json
decide({"runId":"r_inc","workItemId":"w_close","snapshot":"d0","decision":"rejected","note":"Recheck the restored replica, confirm rotated credentials, and correct the customer notice.","requestId":"id-reject"})
```

The rejection is accepted. P:80 says it “moves every branch's completions to history and forks again.” The guaranteed record consequences for the branch data are:

```json
{"ok":true,"data":{"data":{"steps":{"operations":{"history":[{"output":{"recoverySummary":"Service restored; recovery checks passed."},"evidence":[{"kind":"link","ref":"RECOVERY-42"}],"by":"u_ops","at":"2026-10-06T09:25:00Z"}]},"security":{"history":[{"output":{"containmentSummary":"Compromised access revoked; no known remaining exposure."},"evidence":[{"kind":"link","ref":"CONTAINMENT-42"}],"by":"u_security","at":"2026-10-06T09:30:00Z"}]},"communications":{"history":[{"output":{"communicationsSummary":"Customer notice published and current."},"evidence":[{"kind":"link","ref":"NOTICE-42"}],"by":"u_comms","at":"2026-10-06T09:35:00Z"}]}}},"steps":[{"id":"respond","state":"waiting"}]}}
```

These history entries are projections of the old completions, not full invented record schemas. There is no current branch `output`/`evidence` until each branch completes again. `activate.output.bridge` stays current because activation precedes the return target. The barrier is reset; all three branches must rejoin, even if only one actually needed substantive correction (P:63,80).

**First unspecified/inconsistent point:** the run view returned by the accepted rejection. C:126 moves the current fields of “every step completed after the returned-to step” into history and gives them no current value; that includes the just-rejected close review. C:130 simultaneously says “the rejection note is kept in `steps.<id>.note`.” The profile says what happens to branch completions but does not reconcile the close-review record. Consequently, this review cannot promise whether the note is at `data.data.steps.close_review.note`, inside `.history`, or in both. The process's instruction to read the rejection note exposes this gap; it does not repair it.

Additional undefined details at the same re-entry boundary:

- `respond` formerly had an `at`. P:74 says it records “`at` and nothing else”; P:80 archives branch completions, not the parallel completion. It does not say whether `respond.at` disappears during rework, remains the old completion timestamp while the step is waiting, or receives history despite “nothing else.”
- The exact state of the already-decided `close_review` while it waits to be reached a second time is not stated. `null` means “has not been reached” (T:39), which is not true historically; `done` has a different implication from an invalidated current approval. No rewind rule chooses the representation.
- Work-item creation timing has the same qualification as 1a. Fresh branch due clocks restart (C:93), but no generation/iteration marker groups these separate work items and their histories in the tool result.

Conditional on the fork's new human items having been created, each lead next calls exactly `get_run({"runId":"r_inc"})`:

| Lead | Exact fields read and promised content |
|---|---|
| Operations | Its new open item, illustratively `id:"w_ops_2"`, `stepId:"operations"`, `kind:"task"`, the original text/schema/evidence requirements, a current `snapshot:"d1"`, and a new `due`. It reads `data.data.steps.operations.history[0]` for the earlier recovery result; there is no current recovery output. It can also read sibling histories. |
| Security | Corresponding item `w_sec_2`, its requirements/current snapshot/new due, `data.data.steps.security.history[0]`, and sibling histories. There is no current containment output. |
| Communications | Corresponding item `w_comms_2`, its requirements/current snapshot/new due, `data.data.steps.communications.history[0]`, and sibling histories. There is no current communications output. |

At illustrative re-fork time 09:45, each new `due` is 10:45; the real values must be copied from the result. Work-item ID syntax and cross-iteration identity policy are not defined, so these illustrative new IDs are not a promised naming convention. The three reads can share a hash if run data is unchanged. The leads can recover earlier evidence, but cannot rely on one specified location for the commander's fresh rejection note. No automatic undo or corrective publication is implied by restarting a branch.

### Questions the parallel profile leaves unanswered in these traces

This is the list of questions needed to produce the preceding trace, not a request for every imaginable BPM feature:

1. Does a mutating tool return only after all immediately executable server transitions and work-item creations, or may its run view expose intermediate states? What observation boundary does “same write” cover beyond branch readiness and the final join?
2. Are all three human items created atomically with fork readiness, and when are they actually delivered to their groups? The profile guarantees simultaneous readiness, not simultaneous human receipt or action.
3. What precisely is `parallel.at`: fork time, join time, or another event? What happens to the old value when the parallel step is entered again, given “nothing else”?
4. Where is the failed task's note and terminal step record? Is that task finished or cancelled?
5. Which exact error is returned to a token invalidated by run failure/cancellation, and to a decision on a removed work item? Which validation takes precedence over snapshot staleness?
6. Where is the just-rejected close-review note after rewinding? What state does that approval show before re-entry?
7. What distinguishes the second occurrence of a branch/work item in the view beyond whatever IDs the server chooses and separate per-step histories? There is no explicit fork occurrence field.
8. Does “refused once” merely describe one example? Nothing prevents another sibling completion before the retry.
9. How does a person learn the complete `decide` call from their work item without already knowing the contracts? No human action-call field exists.
10. What exact `assignedTo` form represents a direct `person` input? The shared example only specifies a group-shaped example.
11. How do leads obtain this run ID or discover assigned items using only `get_run` and `decide`? Notification/inbox routing and payload are not specified in these contracts.
12. What must clients do to stop external work when another branch ends the run? Server cleanup is specified; a push interruption contract is not.
13. What real-world moment should the one-hour clock measure? The profile inherits readiness as its definition; it supplies no connection to the commander's order, incident declaration, or actual notification receipt.

Questions 1, 4–6, and 9–12 are inherited/shared contract gaps exposed by the parallel scenario, not necessarily obligations that belong solely in the profile. Role membership, substantive approval authority, external recovery, and truth of evidence are explicitly outside server enforcement; this review does not mistake those exclusions for hidden guarantees.

### Do the deadlines measure the commander's expected moment?

Only if the commander expects **branch readiness after accepted activation**. If the incident is declared at 09:00, the bridge opens at 09:02, the agent's submission is accepted and forks at 09:05, the lead is notified at 09:08, and the lead starts at 09:15, `due:1h` means 10:05. It means neither 10:00, 10:02, 10:08, nor 10:15. The clock can be delayed by bridge creation, an expired activation lease, agent retry, or delayed activation submission. Rejection at 09:45 creates a fresh one-hour clock rather than preserving the original incident deadline. The server marks overdue and notifies; work stays open (C:93). The returned due values are useful and explicit, but “one-hour incident response deadline” would overstate this contract.

## Task 2 — Humans restricted to get_run and decide

There are two different tests here. A **contract-aware client** knows T:144–165 and can translate `kind:"approve"` into a decision call. A **person given only a result** does not receive an action description that names `decide` or lists permitted decision values. The following tables distinguish these, rather than calling ordinary user-authored output a protocol defect.

Every human is assumed authenticated under the correct identity and given the run ID. Neither `get_run` nor `decide` discovers that ID for a person with no previous handoff. `get_run` exposes the whole run's open items, not a promised `assignedToMe` flag. Group assignment values do not tell a person their group membership; a direct-person assignment's precise JSON representation is not shown. The server still checks identity and assignment (T:165).

### 2a. Expense reimbursement: receipts and manager approval

Folder: `expense-reimbursement`.

```markdown
---
name: expense-reimbursement
description: Collect an employee's expense receipts and obtain manager approval for reimbursement.
inputs:
  amount: number
  currency: string
  purpose: string
  manager: person
steps:
  - id: receipts
    person: initiator
    task: Describe the expense and attach the receipt files supporting the amount and purpose.
    output:
      expenseSummary: string
    evidence: [file]
    next: manager_review
  - id: manager_review
    person: inputs.manager
    approve: Inspect the receipts and approve only justified expenses within company policy.
    on_reject: receipts
    next: done
  - id: done
    finish: approved
---

# Expense reimbursement

The employee provides receipt files. The manager inspects the actual receipts
against the requested amount, currency, and purpose. Rejection returns the
claim to the employee for correction. Approval records authorization for
reimbursement; payment itself is outside this process.
```

This deliberately asks for actual uploaded receipt evidence, supported by the protocol, rather than quietly replacing receipts with unchecked text. The server must offer `upload` to publish this process (C:206,210). The imposed two-tool human exercise nevertheless excludes calling it.

**Employee's first read:**

```json
get_run({"runId":"r_exp"})
```

Result projection:

```json
{"ok":true,"data":{"run":{"id":"r_exp","state":"waiting"},"data":{"inputs":{"amount":45,"currency":"USD","purpose":"Taxi to customer workshop","manager":"manager@example.org"},"steps":{}},"workItems":[{"id":"w_receipts","stepId":"receipts","kind":"task","text":"Describe the expense and attach the receipt files supporting the amount and purpose.","output":{"expenseSummary":"string"},"evidence":["file"],"next":null,"snapshot":"e0","escalation":null}]}}
```

The employee also reads `data.body`. No file ID or file bytes appear in this fresh run. No tool-result field contains a `decide` recipe.

The exact attempted call that this surface can construct **without inventing a receipt ID** is:

```json
decide({"runId":"r_exp","workItemId":"w_receipts","snapshot":"e0","decision":"completed","output":{"expenseSummary":"Taxi to customer workshop"},"evidence":[],"requestId":"ex-receipts"})
```

Argument derivation for every argument:

| Argument sent | Result field supplying its value | Where its name comes from; missing information |
|---|---|---|
| `runId:"r_exp"` | `data.run.id` | The mapping from `run.id` to `runId` is in the tool contract, not an action field in the result. |
| `workItemId:"w_receipts"` | `data.workItems[id=w_receipts].id` | T:149 supplies the parameter name. The item itself calls it `id`. |
| `snapshot:"e0"` | `data.workItems[id=w_receipts].snapshot` | Both spelling and value are present, but the result does not say to send them to `decide`. |
| `decision:"completed"` | No field contains this value. `kind:"task"` supplies the prerequisite discriminator. | The `decision` name and mapping `task → completed` require C:159 or T:157. Underivable from a promised action recipe because none exists. |
| `output` object | `workItems[].output` is the schema, not a completed output object. | T:151 names the argument; the result supplies its required shape. |
| `output.expenseSummary:"Taxi to customer workshop"` | Name/type: `workItems[].output.expenseSummary`. Value in this example: `data.data.inputs.purpose`. | Copying purpose is the human's chosen factual summary; the schema does not command that copy. |
| `evidence:[]` | `workItems[].evidence:["file"]` supplies the requirement, **not** the empty value. | T:151 names the argument. The empty array is a deliberately insufficient attempted value. A required `{kind:"file",ref:...}` needs a real upload ID. None is available in this read. |
| `requestId:"ex-receipts"` | None. | The name and requirement come from T:5,152/C:184. The client must generate the value. This is intentional client responsibility, not a field the server should necessarily issue. |

Result projection:

```json
{"ok":false,"error":{"code":"not_accepted","issues":["evidence: file required"]}}
```

The particular issue text is one of T:137's examples, not a mandated universal serialization of this error. What is guaranteed is `not_accepted` with issues identifying the evidence failure. Nothing changes. Repeating `get_run({"runId":"r_exp"})` still shows the open receipt item and no manager-review item.

**First blocking point:** the employee cannot turn a new receipt into a valid upload ID using only `get_run` and `decide`. A filesystem path, document number, guessed `f_123`, or bare URL is not a valid substitute for uploaded file evidence (C:226–233). This is a limitation of the imposed two-tool surface, not evidence that the full protocol lacks upload support. If the run already exposed a valid uploaded file reference, the employee could copy its `ref`; this particular fresh process does not.

**Manager's play, explicitly conditional:** if some authorized path outside the two-tool restriction uploads the receipt and the employee validly completes the receipt task, the manager then calls:

```json
get_run({"runId":"r_exp"})
```

Result projection:

```json
{"ok":true,"data":{"run":{"id":"r_exp"},"data":{"inputs":{"amount":45,"currency":"USD","purpose":"Taxi to customer workshop"},"steps":{"receipts":{"output":{"expenseSummary":"Taxi to customer workshop"},"evidence":[{"kind":"file","ref":"f_receipt","sha256":"illustrative-hash","url":"https://files.example.org/receipt?temporary=1"}]}}},"workItems":[{"id":"w_manager","stepId":"manager_review","kind":"approve","text":"Inspect the receipts and approve only justified expenses within company policy.","output":null,"evidence":null,"next":null,"snapshot":"e1","escalation":null}]}}
```

The URL and hash are illustrative returned values. A `file` reference's URL is expressly promised in run data (C:233); a `get_run` result still does not include the actual receipt pixels or text. Opening the link is an external content access, not one of the two tools.

After a real inspection, the manager can send:

```json
decide({"runId":"r_exp","workItemId":"w_manager","snapshot":"e1","decision":"approved","requestId":"ex-manager"})
```

| Argument | Value source in the result | Name/action source |
|---|---|---|
| `runId:"r_exp"` | `data.run.id` | T:149 |
| `workItemId:"w_manager"` | `data.workItems[id=w_manager].id` | T:149 |
| `snapshot:"e1"` | That item's `snapshot` | Field spelling plus T:149 |
| `decision:"approved"` | **No field supplies the value.** The item has `kind:"approve"`; the manager supplies the judgment. | `approve → approved` is learned from T:158, not an action list in the result. |
| `requestId:"ex-manager"` | **No result field.** Client-generated. | T:5,152 |

Accepted result projection, once the finish executes: `{"ok":true,"data":{"run":{"id":"r_exp","state":"ended","outcome":"approved"},"data":{"steps":{"manager_review":{"decision":"approved"}}},"workItems":[]}}`.

A real manager should refuse to approve on the summary alone when the instruction requires inspection of receipts. The protocol's file hash proves that stored bytes match their upload record, not that the amount is reimbursable, the receipt is authentic, or the manager has the appropriate monetary authority. The last point is explicitly a non-goal (C:13). If “only two tools” permits a human to open returned URLs in another application, inspection becomes practical; if it forbids all such access, the manager is also blocked. That distinction must not be hidden.

### 2b. Contract review: legal, finance, signature

Folder: `contract-review`.

```markdown
---
name: contract-review
description: Obtain legal and finance approval, then record execution of the approved contract.
inputs:
  contractRef: string
  amount: number
  currency: string
steps:
  - id: legal_review
    person: legal
    approve: Read the contract at inputs.contractRef and approve its legal terms before it goes to finance.
    next: finance_review
  - id: finance_review
    person: finance
    approve: Read the contract at inputs.contractRef and approve its amount, currency, and financial obligations.
    next: signature
  - id: signature
    person: signer
    task: Sign the contract approved by legal and finance using the organization's signing system, then record the executed document.
    output:
      signedContractRef: string
    evidence: [link]
    next: done
  - id: done
    finish: executed
---

# Contract review

Legal and finance inspect the contract independently in sequence. The signer
must execute the approved contract through the organization's signing system
and attach the resulting executed-document reference. Recording a task
completion in this process is not itself a signature operation.
```

**Legal reviewer:**

```json
get_run({"runId":"r_contract"})
```

Reads `data.body` and:

```json
{"ok":true,"data":{"run":{"id":"r_contract"},"data":{"inputs":{"contractRef":"CONTRACT-77","amount":12000,"currency":"USD"}},"workItems":[{"id":"w_legal","stepId":"legal_review","kind":"approve","text":"Read the contract at inputs.contractRef and approve its legal terms before it goes to finance.","output":null,"evidence":null,"next":null,"assignedTo":{"group":"g_legal"},"snapshot":"k0","escalation":null}]}}
```

After inspecting the contract through its owning system, legal sends:

```json
decide({"runId":"r_contract","workItemId":"w_legal","snapshot":"k0","decision":"approved","requestId":"ct-legal"})
```

Reads `ok:true`, `data.data.steps.legal_review.decision:"approved"`, its `by`/`at`, and the ensuing finance work item when created. No legal output schema or approval evidence field is accepted as part of this approval: those requirements are null on approvals (T:40).

**Finance reviewer:**

```json
get_run({"runId":"r_contract"})
```

Reads inputs, `data.data.steps.legal_review.decision/by/at`, the body, and:

```json
{"ok":true,"data":{"run":{"id":"r_contract"},"workItems":[{"id":"w_finance","stepId":"finance_review","kind":"approve","text":"Read the contract at inputs.contractRef and approve its amount, currency, and financial obligations.","output":null,"evidence":null,"next":null,"assignedTo":{"group":"g_finance"},"snapshot":"k1","escalation":null}]}}
```

After inspecting the actual financial obligations, finance sends:

```json
decide({"runId":"r_contract","workItemId":"w_finance","snapshot":"k1","decision":"approved","requestId":"ct-finance"})
```

Reads `ok:true`, `data.data.steps.finance_review.decision:"approved"`, its `by`/`at`, and the ensuing signature work item when created.

**Signer:**

```json
get_run({"runId":"r_contract"})
```

Reads inputs, both approval records, the body, and:

```json
{"ok":true,"data":{"run":{"id":"r_contract"},"workItems":[{"id":"w_signature","stepId":"signature","kind":"task","text":"Sign the contract approved by legal and finance using the organization's signing system, then record the executed document.","output":{"signedContractRef":"string"},"evidence":["link"],"next":null,"assignedTo":{"group":"g_signer"},"snapshot":"k2","escalation":null}]}}
```

Only after an external signing operation, whose result is illustratively `SIGNED-77`, the signer sends:

```json
decide({"runId":"r_contract","workItemId":"w_signature","snapshot":"k2","decision":"completed","output":{"signedContractRef":"SIGNED-77"},"evidence":[{"kind":"link","ref":"SIGNED-77"}],"requestId":"ct-sign"})
```

Reads `ok:true`, the exact submitted `data.data.steps.signature.output/evidence`, and, once the finish executes, `data.run.state:"ended"`, `outcome:"executed"`, and `workItems:[]`.

The complete argument-provenance table for these three human decisions:

| Argument name | Legal value/source | Finance value/source | Signer value/source |
|---|---|---|---|
| `runId` | `r_contract` from `data.run.id` | Same field/value | Same field/value |
| `workItemId` | `w_legal` from its work item's `id` | `w_finance` from its item's `id` | `w_signature` from its item's `id` |
| `snapshot` | `k0` from its item's `snapshot` | `k1` from its item's `snapshot` | `k2` from its item's `snapshot` |
| `decision` | `approved`: no result field containing that value; `kind:"approve"` plus T:158 and human judgment | Same derivation | `completed`: no result field containing that value; `kind:"task"` plus T:157 and human judgment |
| `output` | Omitted, consistent with item's `output:null` | Omitted for the same reason | Argument name from T:151; field name/type from `workItems[].output.signedContractRef`; value `SIGNED-77` **not derivable from any returned field**. It is produced by external signing. |
| `evidence` | Omitted, consistent with item's `evidence:null` | Omitted for the same reason | Required kind `link` from `workItems[].evidence[0]`; object keys `kind`/`ref` from T:47. `ref:"SIGNED-77"` is externally produced, not returned by `get_run`. |
| `requestId` | `ct-legal`: client-generated, no result source | `ct-finance`: client-generated, no result source | `ct-sign`: client-generated, no result source |

For every column, the parameter names `runId`, `workItemId`, `decision`, and `requestId` are learned from the tool contract, not from a result-carried action. `snapshot` and `output` happen to have matching field names in both places, but their required use still comes from the contract. None of these calls sends `next` or `reason`; each process route is scalar, and approvals cannot take a route choice at all.

If any approver instead declines, the tools document supplies `decision:"rejected"` and mandatory `note`; the work item does not list that option or the note requirement. These particular approvals have no `on_reject`, so a rejection ends the run `rejected` (C:92,161). A documents-ignorant person cannot learn that consequence from `workItems[].next:null`.

**What the humans should refuse to treat as sufficient:**

- Legal should not approve a bare `CONTRACT-77` string as though the contract's contents had been returned. `get_run` supplies a reference and instructions, not document retrieval for arbitrary link evidence or inputs.
- Finance should not substitute the scalar `amount` for inspection of the contract's complete financial obligations. The input schema verifies the type, not consistency with the actual contract.
- The signer should not call `decide(completed)` as a signing mechanism. These tools neither execute an e-signature nor return proof of one. `SIGNED-77` must come from real work outside the two-tool surface.
- All three need confidence that the document reviewed and signed is the same version. A pinned **process** version (C:241) does not pin the contents behind a mutable `contractRef`. Snapshot checks cover run data; an external document can change while its reference string and the snapshot stay unchanged. An immutable document reference supplied by the owning system can satisfy organizational practice, but immutability is not guaranteed by `string` or `link` here.

The ordinary workflow acknowledgements can be executed with the two tools by a contract-aware client. The actual receipt upload, document inspection, and signature cannot be performed solely through those two tools. That boundary is legitimate for an orchestrator, but claiming that people can complete the underlying work using *only* the result fields would be false. User-authored outputs and generated request IDs are naturally underivable from prior results; missing human action descriptions are a separate discoverability issue.

## Task 3 — Retry, race, event, cancellation, and initiator edge cases

### 3a. Submission succeeds, response is lost, lease expires, same requestId is retried

Assume activation was validly claimed:

```json
claim({"runId":"r_replay","stepId":"activate","requestId":"re-claim"})
```

Result projection: `{"ok":true,"data":{"token":"token-replay","expiresAt":"2026-10-06T09:10:00Z","mode":"live"}}`. The agent reads its instructions and schemas too, as in 1a.

At 09:05 it sends:

```json
submit({"token":"token-replay","output":{"bridge":"BRIDGE-REPLAY"},"evidence":[{"kind":"link","ref":"BRIDGE-REPLAY"}],"requestId":"re-submit"})
```

The server accepts it. The original success result is a run view, illustratively with `ok:true`, `data.steps[id=activate].state:"done"`, and `data.data.steps.activate.output.bridge:"BRIDGE-REPLAY"`. The client receives **no result** because the response is lost. It must not claim to have observed the success then.

At 09:11, after the old lease expiry, it sends **exactly**:

```json
submit({"token":"token-replay","output":{"bridge":"BRIDGE-REPLAY"},"evidence":[{"kind":"link","ref":"BRIDGE-REPLAY"}],"requestId":"re-submit"})
```

The server returns the **original successful run view**, not `stale`, with the same fields and values as that lost result (C:145,184; T:133). The actor reads `ok:true` and the completion record; no new completion is created. The original run view can now be old: branch completions after 09:05 need not appear in it. To observe current state, it sends:

```json
get_run({"runId":"r_replay"})
```

and reads the current `data.run.state`, `data.steps`, `data.workItems`, and run data. Their actual values depend on intervening actors, which this scenario does not specify.

**Settled:** yes, for protocol state and same-argument replay. Idempotent success must be found before treating the now-invalid old token as a fresh submission. C:145's “token included” and “reads `expiresAt`” apply to replayed **claim** results; `submit` returns a run view and has no promised `expiresAt`. The replay guarantee must not be interpreted as a new lease. It also does not deduplicate external bridge creation if the agent repeats its own external actions unnecessarily. No unspecified point prevents this trace.

### 3b. Stale human decision recovers to success

The concrete calls and results are 1b, including both reads, the sibling decision, stale refusal, fresh read, and accepted retry. To make the required recovery signature unmistakable:

```json
decide({"runId":"r_inc","workItemId":"w_sec","snapshot":"b0","decision":"completed","output":{"containmentSummary":"Access revoked; containment verified."},"evidence":[{"kind":"link","ref":"CONTAINMENT-42"}],"requestId":"ib-sec"})
```

returns the error projection `{"ok":false,"error":{"code":"stale"}}` after the sibling's completion. Then:

```json
get_run({"runId":"r_inc"})
```

returns the still-open work item with `snapshot:"b1"` and the updated sibling record. Finally:

```json
decide({"runId":"r_inc","workItemId":"w_sec","snapshot":"b1","decision":"completed","output":{"containmentSummary":"Access revoked; containment verified."},"evidence":[{"kind":"link","ref":"CONTAINMENT-42"}],"requestId":"ib-sec"})
```

returns `ok:true`, records `data.data.steps.security.output/evidence/by/at`, marks the step done, and removes that work item if nothing changes between the fresh read and this call. The decision value is reconsidered by the person, not blindly mandated to stay the same after learning the sibling result.

**Settled:** yes, conditionally on the item still existing and no further change. A failed call's request ID is not recorded, so changing the snapshot does not conflict with an already-recorded request. Recovery to success is not guaranteed if another actor decides/removes the item or the run terminates; `get_run` must be inspected, not merely mined for a newer hash.

### 3c. Two same-name events before readiness; a third after consumption

Use this complete small process to make “after consumed” unambiguous: there is exactly one matching wait, and the run remains open afterward.

```markdown
---
name: event-review
description: Wait for one external proof delivery, then have a reviewer inspect it.
steps:
  - id: prepare
    agent: Prepare the case for receiving external proof.
    next: proof
  - id: proof
    wait_for: proof_received
    next: review
  - id: review
    person: reviewer
    approve: Inspect the proof event recorded in steps.proof.output.
    next: done
  - id: done
    finish: reviewed
---

# Event review

Preparation precedes the proof wait. The reviewer inspects the single
consumed proof delivery before concluding the process.
```

After a successful `start_run({"process":"event-review","inputs":{},"requestId":"ev-start"})`, the agent reads `data.run.id:"r_events"` and `prepare` ready. It sends:

```json
claim({"runId":"r_events","stepId":"prepare","requestId":"ev-claim"})
```

and reads `data.token:"token-events"`, its expiry, and its no-output/no-evidence requirements. Before submitting preparation:

```json
send_event({"runId":"r_events","name":"proof_received","data":{"delivery":1},"requestId":"ev-one"})
```

returns exactly the result shape `{"ok":true,"data":{"consumedBy":null}}`.

```json
send_event({"runId":"r_events","name":"proof_received","data":{"delivery":2},"requestId":"ev-two"})
```

also returns `{"ok":true,"data":{"consumedBy":null}}`. These are two distinct deliveries because their request IDs differ. Their same event name does not deduplicate them. They are held oldest first (C:178; T:179).

The agent then sends:

```json
submit({"token":"token-events","output":{},"requestId":"ev-prepare-done"})
```

After the ready wait consumes its oldest held event, `get_run({"runId":"r_events"})` returns the relevant projection:

```json
{"ok":true,"data":{"run":{"id":"r_events","state":"waiting"},"data":{"steps":{"proof":{"output":{"delivery":1}}}},"steps":[{"id":"proof","state":"done"},{"id":"review","state":"waiting"}],"workItems":[{"id":"w_event_review","stepId":"review","kind":"approve"}]}}
```

`delivery:1` is settled by FIFO. Delivery 2 does not overwrite the completed wait's output or satisfy that same occurrence a second time. No queue-inspection field is promised in the run view. The retained/unused delivery is not visible there.

While review is still open, a sender makes the third delivery:

```json
send_event({"runId":"r_events","name":"proof_received","data":{"delivery":3},"requestId":"ev-three"})
```

**First unspecified point for this event sequence:** the result and lifecycle of this fresh event when the only matching wait has already completed and there is no future matching wait. C:178 guarantees holding for arrivals *before* a wait becomes ready. T:179 defines `consumedBy:null` when held “for a wait not yet ready.” Neither expressly says what happens to an event for an exhausted wait name. General “holds events per run” suggests accepting and retaining it; that reading yields `{"ok":true,"data":{"consumedBy":null}}`. That is a plausible **conditional** result, not a guaranteed exact response for this post-consumption case. Rejecting an unconsumable fresh event is not given a specific error contract either. Cleanup/retention of the already-held second delivery with no future consumer is also undefined.

If a later matching wait had actually been declared, FIFO would require delivery 2 to satisfy it before delivery 3. If a wait were currently ready with no held older event, a newly delivered event would return that wait's step ID as `consumedBy`. Neither condition applies to this single-wait process. Replaying `ev-one` with identical arguments after consumption would instead return its original `consumedBy:null`, not a retrospectively updated delivery receipt, because write replay returns the original result (C:184).

**Settled:** initial buffering, distinct deliveries, FIFO, and recorded data are settled. Fresh arrivals after the final matching wait, unused-event retention, and a delivery-status query are not. Automatic-step response timing has the already-identified 1a qualification; the event-specific gap is the third delivery.

### 3d. Operator cancels with an agent claim and an open human item

Use the already-defined mixed incident variant, with run ID `r_cancel`: communications holds `token-cancel-comms`, security has open item `w_cancel_sec` with previously read snapshot `z0`, and operations has another open item. The claim and work-item prerequisites are the same calls as 1c with these illustrative IDs.

Operator sends:

```json
cancel({"runId":"r_cancel","reason":"Incident handling moved to the disaster-recovery process.","requestId":"ca-cancel"})
```

Guaranteed result projection:

```json
{"ok":true,"data":{"run":{"id":"r_cancel","state":"cancelled"},"steps":[{"id":"respond","state":"cancelled"},{"id":"operations","state":"cancelled"},{"id":"security","state":"cancelled"},{"id":"communications","state":"cancelled"}],"workItems":[]}}
```

`outcome:"cancelled"` is **not** promised, so it is not fabricated. The reason is a required input, but no result field is specified for its stored location. Earlier completed activation remains completed; cancellation does not undo its external bridge or pages (C:174).

The claimed agent next tries:

```json
submit({"token":"token-cancel-comms","output":{"communicationsSummary":"Notice published."},"evidence":[{"kind":"link","ref":"NOTICE-CANCEL"}],"requestId":"ca-agent-late"})
```

The token is invalid and the call must not complete the cancelled step. An exact error code is not unambiguously promised for terminal invalidation, as explained in 1c. The agent then calls `get_run({"runId":"r_cancel"})`; it reads `run.state:"cancelled"`, its step `cancelled`, and `workItems:[]`. If it calls `get_work({"process":"major-incident-agent-comms"})`, this run contributes no ready item; other runs may still appear.

The security person next tries:

```json
decide({"runId":"r_cancel","workItemId":"w_cancel_sec","snapshot":"z0","decision":"completed","output":{"containmentSummary":"Containment completed."},"evidence":[{"kind":"link","ref":"CONTAINMENT-CANCEL"}],"requestId":"ca-human-late"})
```

The removed item cannot be completed. The specific error is undefined for the removed-by-cancellation case. Notably, the snapshot is a hash of **run data**, not the entire run view (C:166): cancellation changes run state and removes items, but no cancellation field is promised inside `data`. It is therefore unsafe to assert that cancellation necessarily changes this snapshot and hence necessarily produces `stale`. `conflict/"already_decided"` is promised for already-decided work, not explicitly for removed undecided work (T:165).

The person then calls `get_run({"runId":"r_cancel"})`; it shows `cancelled`, their step cancelled, and no work items. Operations, the commander, and the original activation agent each see that same terminal state on their next `get_run({"runId":"r_cancel"})`; no commander item is created. The operator's next `get_run({"runId":"r_cancel"})` confirms cancellation but has no guaranteed cancellation-reason field to display.

**First unspecified point:** the accepted cancellation result's representation of the cancellation reason, if the operator needs it in the record; more consequentially, the exact error on each participant's subsequent write. **Settled:** terminal cleanup and nonacceptance of late work. **Not settled:** complete terminal audit fields, exact late-call error behavior/precedence, and proactive interruption of external work. Cancelling an already ended run is expressly `conflict` (T:169); that separate rule does not fill these gaps.

### 3e. Agent first step, later initiator task, started by an agent

Complete process:

```markdown
---
name: initiator-check
description: Prepare a result and ask the person who initiated the run to confirm it.
steps:
  - id: prepare
    agent: Prepare the result for the initiating person.
    output:
      summary: string
    next: confirm
  - id: confirm
    person: initiator
    task: Confirm that the prepared result meets your request.
    output:
      confirmation: string
    next: done
  - id: done
    finish: confirmed
---

# Initiator check

A person must start this process because confirmation belongs to that person.
```

An authenticated agent sends:

```json
start_run({"process":"initiator-check","inputs":{},"requestId":"in-agent-start"})
```

Guaranteed error projection:

```json
{"ok":false,"error":{"code":"invalid"}}
```

The required `message` text and any `issues` strings are unspecified. The server refuses at start because **any** step assigned to `initiator` triggers the agent-start restriction (T:70; C:153), even though the first step is agent work. There is no successful run view or new run ID, and the first agent step is not made available by this refused start. The client must not guess a run ID and proceed to claim.

**Settled:** yes. No lazy runtime resolution or later deadlock is required here. The server can see the prohibited assignment in the process at start. The actor cannot rescue this same agent-start call by supplying a human email as an unrelated input: the reserved `initiator` semantics use the starter's identity, not arbitrary inputs.

## Task 4 — What to remove from the parallel profile

These are removals or scope reductions, not proposals for new features:

1. **Remove “refused once” from P:82.** It promises a bound the snapshot rules do not provide. The remaining read-current-state/refuse-stale rule already explains the actual behavior.
2. **Remove the endorsement “That is the intended behaviour” in P:82.** It treats an across-branch human conflict as a benefit without specifying any bounded completion property. The normative core already defines snapshot validity; the profile does not need to sell the resulting operational friction.
3. **Remove the parallel step's special run-data record, “records `at` and nothing else” (P:74), unless its repeated-execution meaning is already fully specified elsewhere.** In these three documents it is not. The barrier's state already appears in `steps[]`, and branch completions already have timestamps. This exceptional record creates the most avoidable history inconsistency in the profile.
4. **Remove the blanket claim that branch steps run “exactly as in the core” (P:70) and the claim that “Nothing else in the tool contracts changes” (P:86).** The profile changes reachability, permits `join` instead of an existing step ID, creates multiple simultaneously pending activities, adds a new kind, and changes rewind scope. The narrower, concrete fork/join/view rules are useful without the blanket equivalence claim.
5. **For a minimum defensible version 1, remove post-join rejection back into a parallel region (P:63 and that portion of P:80), including the example's `on_reject: respond`.** This is a substantive loss of functionality, not cosmetic editing: it removes the exact rework feature whose run-view/history semantics the supplied contracts do not complete. If this requested incident-rework capability remains a product requirement, this removal is not an acceptable substitute for fulfilling that requirement; the current draft must instead remain unclaimed as complete. The review is not proposing a replacement retry feature.

Keep the simultaneous readiness rule, explicit all-branches join, disjoint branch validation, restriction against nesting, and terminal cleanup. Removing those would erase the actual value of the profile or weaken correctness. Do not add thresholds, nested parallelism, selective branch retry, compensation, or a new policy language just to answer this review; none is needed to establish the existing contract's limits.

## Verdict

For an organization's major incidents, this draft is a credible small orchestration primitive, but not yet a complete production operating contract: it can open three independent work items, collect schema-valid evidence references, and gate commander review on all three completions; its readiness deadlines, across-branch stale decisions, terminal reply gaps, and ambiguous rework records materially affect how responders operate under pressure. An agent plus three humans who are ignorant of the documentation cannot be promised to complete it from tool results alone: the agent gets an explicit claim recipe, while humans get schemas and opaque identifiers without decision-call affordances, need an out-of-band run-ID handoff and contract-aware client, and still need their external systems for receipts, document inspection, recovery, publication, and signing. The protocol explicitly disclaims business authority and correctness, so it should be judged as a coordination mechanism; even on that narrower basis, the stated edge cases expose concrete unfinished contracts rather than merely missing enterprise extras.
