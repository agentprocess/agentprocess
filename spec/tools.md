---
title: "Tool contracts"
description: "The exact arguments, results and errors of every tool. Part of the specification."
---

Version `core-2`. Companion to the [specification](specification.md) §4. These shapes are normative. Types use the field notation of the core: a bare type name, `?` for optional, `[]` for lists.

Every result is `{ ok: true, data: <shape below> }` or `{ ok: false, error: { code, message, issues?: string[] } }`. All writes take `requestId: string`.

JSON Schemas (draft 2020-12) for every tool's arguments and result and for the shared shapes are in [`schemas/core-2/`](../schemas/core-2/index.json); `index.json` lists them and their `$id`s are under `https://agentprocess.io/schemas/core-2/`. Where a schema and this file disagree, this file wins and the schema is a bug. The agent skill that teaches the loop is [skills/agentprocess](../agents/skill.md).

## Shared shapes

**Run view** (returned by `start_run`, `get_run`, `submit`, `decide`, `escalate`, `cancel`):

```json
{
  "run": { "id": "r_8f2", "process": "supplier-onboarding", "version": 3, "mode": "live",
           "state": "waiting", "outcome": null, "ended": null, "startedBy": "u_mira", "startedAt": "2026-10-06T09:00:00Z",
           "updatedAt": "2026-10-06T09:04:12Z" },
  "data": {
    "inputs": { "vendorName": "Acme", "amount": 48000 },
    "steps": {
      "check_vendor": {
        "output": { "cleared": true, "summary": "No matches on OFAC or EU lists." },
        "summary": "Searched both lists by legal name and two known aliases. No hits. Report attached. Did not check beneficial owners; not in scope.",
        "evidence": [ { "kind": "file", "ref": "f_91a", "sha256": "4f1c…", "url": "https://…/files/f_91a?x=…" } ],
        "next": "approve", "reason": "Vendor is cleared.",
        "by": "a_screener", "at": "2026-10-06T09:04:12Z",
        "history": [] } } },
  "steps": [
    { "id": "check_vendor", "kind": "agent", "state": "done" },
    { "id": "approve", "kind": "approve", "state": "waiting", "due": "2026-10-08T09:04:12Z", "overdue": false },
    { "id": "decline", "kind": "finish", "state": null },
    { "id": "done", "kind": "finish", "state": null } ],
  "workItems": [
    { "id": "w_31", "stepId": "approve", "kind": "approve", "text": "Approve only when the vendor is cleared and the spend is justified.",
      "output": null, "evidence": null, "next": null, "handoff": null,
      "assignedTo": { "group": "g_finance" }, "snapshot": "9b7e…", "due": "2026-10-08T09:04:12Z",
      "escalation": null,
      "decide": { "tool": "decide", "arguments": { "runId": "r_8f2", "workItemId": "w_31", "snapshot": "9b7e…", "requestId": "<new UUID>" },
                  "decisions": ["approved", "rejected"] } } ],
  "body": "# Supplier onboarding\n\nStart this when …"
}
```

- `steps[].state` is `null` for a step that is not currently reached. A step rewound by a rejection is `null` and keeps its `history`. `failed` marks the step whose `failed` decision ended the run; its `note`, `by` and `at` are in run data.
- `run.ended` is `null` while the run is active or waiting, and `{ "by": "u_ops", "at": "…", "note": "…" }` once it ended or was cancelled: the cancel reason, the `failed` note, or `note: null` for a finish.
- `workItems[].assignedTo` is `{ "group": "<id>" }` for a role, or `{ "person": <person value> }` for `initiator` or a `person` path.
- `workItems[].handoff` is the same object `claim` returns: what the previous attempt left, or `null`. A task returned by an operator carries `handoff.returned`; a task bounced by a check carries `handoff.check`.
- `workItems[].escalation`, when a check was unsure, also carries `check` (the verdict) and `candidate` (the held submission); `completed` without `output` uses the candidate.
- `workItems[].decide` is the exact call to make, with `decisions` listing what this item accepts: `["completed", "failed"]` on a task, `["approved", "rejected", "failed"]` on an approval, `["returned", "completed", "failed"]` on an escalated agent step. The person fills `requestId` and adds `decision` and whatever that decision requires.
- `workItems[].output`, `evidence` and `next` are the step's requirements exactly as `claim.step` gives them: the output fields, the required evidence kinds, and the routes `[{ to, when? }]` when `next` is a list. They are `null` on an approval, which takes none of them.
- `workItems[].snapshot` is computed at the time of this read. A later read of the same item may return a different value.
- `workItems[].escalation` is `{ "note": "…", "by": "a_screener", "at": "…" }` on an escalated agent step, else `null`.
- `data.steps.<id>` for an approval has `decision`, `note`, `by`, `at`, `history` and nothing else. After a rejection it keeps these current until the approval completes again.
- `data.steps.<id>` for the step whose `failed` decision ended the run has `decision: "failed"`, `note`, `by`, `at`.
- `data.steps.<id>` for a `wait_for` step has `output` set to the event `data`.
- In `mode: "test"`, every work item and `get_work` item also carries `"mode": "test"`.

**Evidence item** (in `submit`, `decide`): `{ "kind": "file" | "link", "ref": string }`. The server adds `sha256` and `url` to `file` items in run data.

## describe

Arguments: none.

```json
{ "protocol": "agentprocess", "version": "core-2", "profiles": ["parallel-1", "check-1"], 
  "leaseSeconds": 600, "person": { "format": "email" }, "tools": ["describe", "list_processes", "…"] }
```

## list_processes

Arguments: `{ cursor?: string }`. Paged like `get_work`.

```json
{ "items": [ { "name": "supplier-onboarding", "description": "Check a new supplier …", "version": 3,
               "inputs": { "vendorName": "string", "amount": { "type": "number", "description": "Expected annual spend in USD" } } } ],
  "nextCursor": null }
```

## start_run

Arguments: `{ process: string, version?: number, inputs: object, mode?: "live" | "test", requestId }`. `version` defaults to the latest published. Undeclared or mistyped inputs → `invalid` with `issues`. A process with any step assigned to `initiator`, when the caller is an agent identity → `invalid`.

Result: the run view.

## get_run

Arguments: `{ runId: string }`. Result: the run view.

## get_work

Arguments: `{ limit?: number (1–100, default 20), cursor?: string, process?: string }`.

Called by an agent identity:

```json
{ "items": [ { "runId": "r_8f2", "stepId": "check_vendor", "process": "supplier-onboarding", "version": 3,
               "mode": "live", "readySince": "2026-10-06T09:00:00Z", "due": null, "overdue": false,
               "handoff": null,
               "claim": { "tool": "claim", "arguments": { "runId": "r_8f2", "stepId": "check_vendor", "requestId": "<new UUID>" } } } ],
  "nextCursor": null }
```

- Ordered by `readySince`, oldest first.
- `handoff` is `null` on a first attempt, else what the previous attempt left: `{ "progress": "…" }` after an expired lease, `{ "returned": { "note": "…", "by": "u_ops", "at": "…" } }` after a person sent it back, `{ "issues": ["…"] }` after a submission was not accepted, `{ "check": { … } }` after a check sent it back (check profile). Several may be present.
- Lists only steps the caller may claim. A server MAY filter further by its own agent configuration; it MUST NOT list a step the caller cannot claim.

Called by a person:

```json
{ "items": [ { "runId": "r_8f2", "process": "supplier-onboarding", "version": 3, "mode": "live",
               "workItem": { "id": "w_31", "stepId": "approve", "kind": "approve", "text": "…", "output": null, "evidence": null, "next": null,
                             "assignedTo": { "group": "g_finance" }, "snapshot": "9b7e…", "due": "…", "escalation": null,
                             "decide": { "tool": "decide", "arguments": { "runId": "r_8f2", "workItemId": "w_31", "snapshot": "9b7e…", "requestId": "<new UUID>" },
                                         "decisions": ["approved", "rejected", "failed"] } } } ],
  "nextCursor": null }
```

- Lists the open work items whose assignment includes the caller, oldest first. `workItem` is the same object `get_run` returns.

## claim

Arguments: `{ runId, stepId, requestId }`.

```json
{ "token": "eyJ…", "expiresAt": "2026-10-06T09:10:00Z", "mode": "live",
  "step": { "id": "check_vendor", "instructions": "Check the vendor against …",
            "output": { "cleared": "boolean", "summary": "string" }, "evidence": ["file"],
            "next": [ { "to": "approve", "when": "Vendor is cleared on both lists" }, { "to": "decline", "when": "Any sanctions hit" } ] },
  "handoff": null,
  "body": "# Supplier onboarding\n\n…",
  "data": { "inputs": { … }, "steps": { … } } }
```

`extensions?: object` carries what a server-specific profile adds for this step, keyed as that profile defines. A client that does not know a key ignores it.

Errors: `conflict` with `issues: ["claimed_by_other"]` when another caller holds a live claim, `["claimed_by_you"]` when the caller does; `conflict` with `["not_ready"]` otherwise; `forbidden` for a non-agent step; `invalid` for a token that does not verify, on any tool that takes one.

Replay with the same `requestId`: the original result, even if the lease has since ended; read `expiresAt`.

## renew

Arguments: `{ token, progress?: string, requestId }`. `progress` is kept on the step while claimed and handed to the next claimant if this lease expires.

```json
{ "token": "eyJ…new", "expiresAt": "2026-10-06T09:20:00Z" }
```

The previous token is invalid once this returns. After the lease ended: `stale`.

## upload

Arguments: `{ fileName: string, contentType: string, contentBase64: string, requestId }`. Size limit from `describe.maxUploadBytes` (default 10 MiB).

```json
{ "id": "f_91a", "sha256": "4f1c…", "bytes": 182311 }
```

## submit

Arguments: `{ token, output: object, evidence?: EvidenceItem[], summary: string, next?: string, reason?: string, requestId }`. `summary` is 1–2000 characters.

Result: the run view. `ok: true` means the submission was accepted: the step is done, or, when a check was unsure, the step is waiting on a person and the view shows the escalation work item (check profile).

Errors:

- `not_accepted`, `issues` such as `"summary: missing"`, `"output.summary: missing"`, `"output.cleared: expected boolean"`, `"output.extra: not declared"`, `"evidence: file required"`, `"evidence f_000: not found"`, `"next: required, one of approve, decline"`, `"reason: required"`. Nothing changed; the claim stands.
- `stale` when the token's lease ended or the claim was replaced.
- `conflict` with `issues: ["run_ended"]` when the run has ended or been cancelled, checked before anything else.

## escalate

Arguments: `{ token, note: string, requestId }`. Result: the run view. The step is `waiting`; a work item exists with `escalation` set; the token is invalid.

## decide

Arguments:

```
{ runId, workItemId, snapshot,
  decision: "completed" | "approved" | "rejected" | "returned" | "failed",
  output?: object, evidence?: EvidenceItem[], summary?: string, next?: string, reason?: string, note?: string,
  requestId }
```

| decision | Allowed on | Requires |
|---|---|---|
| `completed` | task, escalated agent step | `output`, `evidence` per the step; `next` and `reason` when the step's `next` is a list |
| `approved` | approve | nothing |
| `rejected` | approve | `note` |
| `returned` | escalated agent step | `note` |
| `failed` | any | `note` |

`next` and `reason` on any decision other than `completed` → `invalid`; on `completed` for a step whose `next` is not a list → `not_accepted`. `note` with `completed` → `invalid`; `output`, `evidence` or `summary` with anything but `completed` → `invalid`.

Result: the run view. Errors, in the order checked: `conflict` with `issues: ["run_ended"]` when the run has ended or been cancelled; `conflict` with `issues: ["already_decided"]` when the item was decided; `not_found` when it never existed on this run; `stale` when `snapshot` is not current, in which case `get_run` returns a fresh one and the same decision may be sent again; `forbidden` for a non-human identity or a person outside the assignment; `not_accepted` as for `submit`.

## cancel

Arguments: `{ runId, reason: string, requestId }`. Result: the run view with `state: "cancelled"` and `run.ended: { by, at, note: reason }`. On an ended run: `conflict` with `issues: ["run_ended"]`.

## send_event

Arguments: `{ runId, name: string, data?: object, requestId }`. From an agent identity or an operator; `forbidden` otherwise.

```json
{ "consumedBy": "settlement_event" }
```

`consumedBy` is `null` when the event is held: for a wait not yet ready, or for no wait at all. A held event with no matching wait stays held until the run ends and is then discarded. Held events with the same name are consumed oldest first. On an ended run: `conflict` with `issues: ["run_ended"]`.

## list_runs

Arguments: `{ process?: string, state?: string, inputs?: object, limit?: number, cursor?: string }`. `inputs` matches on equality of the given fields.

```json
{ "items": [ { "id": "r_8f2", "process": "supplier-onboarding", "version": 3, "state": "waiting", "outcome": null,
               "startedAt": "…", "updatedAt": "…", "inputs": { "vendorName": "Acme", "amount": 48000 } } ],
  "nextCursor": null }
```
