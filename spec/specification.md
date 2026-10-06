---
title: "Specification"
description: "The PROCESS.md format, how a server runs it, and the five things a server guarantees."
---

Version `core-2`, draft, 6 October 2026. Tool contracts: [tools](tools.md), part of this specification. Profiles: [parallel](profiles/parallel.md), [check](profiles/check.md). History: [revisions](revisions.md). JSON Schemas: [`schemas/core-2/`](../schemas/core-2/index.json).

The key words MUST, MUST NOT, SHOULD and MAY are to be read as in RFC 2119.

Design rule, borrowed from [Agent Skills](https://agentskills.io): **the format carries as little as possible; the instructions and the model carry the rest.** A server enforces five things (§6). Everything scenario-specific is plain language in a step that the agent or person adapts to. The one place this rule does not apply is the wire between agent and server: an agent can fill a gap in instructions, it cannot fill a gap in a tool contract, so the tool contracts are exact.

---

## 1. What it is

An **agentprocess** is one `PROCESS.md` file: a short frontmatter that names the process, its inputs and its steps, and a body in plain language. A **process server** runs it: it hands steps to agents and people in order, accepts their output, and keeps the record. An **agent** is any program that connects to the server over MCP, claims a step, does it with its own tools, and submits the result.

A process file enforces nothing on its own. It is enforced when a server runs it.

Non-goals: a server does not enforce approval authority matrices, reporting lines, segregation of duties, or the business correctness of an output. Instructions say what the right answer is; people and organization policy check it. A server enforces only §6.

## 2. PROCESS.md

```markdown
---
name: supplier-onboarding
description: Check a new supplier against sanctions lists and get finance approval.
inputs:
  vendorName: string
  amount: { type: number, description: Expected annual spend in USD }
steps:
  - id: check_vendor
    agent: |
      Check the vendor against the OFAC and EU sanctions lists.
      Attach the screening report.
    output:
      cleared: boolean
      summary: string
    evidence: [file]
    next:
      - { to: approve, when: Vendor is cleared on both lists }
      - { to: decline, when: Any sanctions hit }

  - id: approve
    person: finance-approver
    approve: Approve only when the vendor is cleared and the spend is justified.
    due: 2d
    on_reject: check_vendor
    next: done

  - id: decline
    finish: declined

  - id: done
    finish: onboarded
---

# Supplier onboarding

Start this when procurement has a new supplier. The agent screens the vendor,
finance approves, and the supplier is onboarded. If finance rejects, the agent
re-checks with the finance note and tries again.
```

The body is for people and agents. A server returns it with every claim and every run view. A catalog for people shows `name` and `description`; `list_processes` also returns `inputs`.

### 2.1 Frontmatter

| Field | Required | Meaning |
|---|---|---|
| `name` | yes | 1–64 chars matching `^[a-z0-9]+(-[a-z0-9]+)*$`. When a process travels as a folder, the folder has this name; an importer given a folder refuses a mismatch. |
| `description` | yes | One or two sentences: what the process does and when to start it. |
| `inputs` | no | Fields a run starts with (§2.3). Default: none. |
| `steps` | yes | Ordered list of steps (§2.2). At least one. |

Unknown fields anywhere in the frontmatter are refused. Frontmatter is YAML 1.2; a server MUST parse it with the core schema only, so `yes` and `no` are strings. Duplicate keys are refused.

### 2.2 Steps

A step has an `id` and exactly one **kind** key. Every other key is optional.

| Kind key | Who completes it | Value |
|---|---|---|
| `agent:` | An agent | Instructions. |
| `task:` | A person | Instructions. Needs `person`. |
| `approve:` | A person | What they are approving. Needs `person`. |
| `wait:` | The server | A duration (§2.4). |
| `wait_until:` | The server | A path into run data holding a `datetime`, e.g. `inputs.startDate`. A time already past continues at once. A missing or invalid value fails the run. |
| `wait_for:` | The server | An event name. Continues when `send_event` delivers it (§3.6). |
| `finish:` | The server | The run's outcome, a short word like `onboarded`. |

Optional keys:

| Key | Applies to | Meaning |
|---|---|---|
| `output` | agent, task | Fields the submission must contain (§2.3). Default: none. |
| `evidence` | agent, task | Evidence kinds that must be attached: `file` or `link` (§5). Default: none. |
| `next` | all but finish | Where the run goes next. A step id, or a list of two or more routes the actor chooses from (§3.2). A route is a step id, or `{ to: <id>, when: <text> }` where `when` says in plain language when to take it; it labels the edge on a process map and guides the actor, and is not enforced. Default: the following step in the list. A list is allowed only on `agent` and `task`; an approval approves or rejects, and a person who must choose a route does it in a `task`. |
| `person` | task, approve | Who: a **role name** local to this process (`finance-approver`), `initiator` (the person who started the run), or a path to a `person` value in run data (`inputs.managerEmail`). A value containing a dot is a path; role names cannot contain dots. A path is `inputs.<field>` or `steps.<id>.output.<field>` and must name a declared `person` field; the same rule applies to `wait_until` with a `datetime` field. |
| `on_reject` | approve | Step to return to on rejection. Default: the run ends with outcome `rejected`. |
| `due` | agent, task, approve | A duration after the step becomes ready. When it passes the server marks the step overdue and notifies the assignee and the organization's operators. The work stays open. A repeat of the step restarts the clock. |
| `timeout` | wait_for | A duration. Default: none. |
| `on_timeout` | wait_for | Step to continue at when `timeout` passes. Default: the step's `next`. |

Step ids match `^[a-z][a-z0-9_-]{0,63}$` and are unique. Every `next`, `on_reject` and `on_timeout` names an existing step. Every step is reachable from the first step; a non-finish step that is last in the list MUST name `next`; a `finish` has no `next`. Following `next` and `on_timeout` edges only, no step is visited twice; `on_reject` is the only edge that goes back. A server MUST refuse a file that breaks any of these.

### 2.3 Fields

`inputs` and `output` map a field name to a type, or to an object with `type` and optional `description`, `optional: true`, `one_of: [...]`, and `items` for lists. `items` is a type name or a field object.

Types: `string`, `number`, `boolean`, `date` (`YYYY-MM-DD`, a real calendar date), `datetime` (RFC 3339 with offset), `list`, `object`, `person` (an identity reference the server resolves; its form is server-defined and shown in `describe`). Numbers are JSON numbers; a server MUST NOT round them. That is the whole schema language. Anything finer goes in the instructions, and the agent follows it.

Every field is required unless `optional: true`. An optional field may be omitted; `null` is not a value. A value outside `one_of` is refused; `one_of` values have the field's type. Fields not declared in `inputs` or `output` are refused.

Run data is visible to everyone who works a step (§3.1). Keep identity documents, bank details and secrets in the systems that own them and put references in the run.

### 2.4 Durations

An integer followed by `m`, `h` or `d`: `30m`, `4h`, `3d`. Nothing else.

## 3. Running

### 3.1 Run data

A run has `inputs` and, for each completed step, `steps.<id>` with:

| Field | Set by |
|---|---|
| `output`, `evidence`, `summary`, `next`, `reason` | A completed agent or task step. `summary` is what was done and found, in the submitter's words. |
| `decision` (`approved` or `rejected`), `note` | A completed approval. |
| `decision: failed`, `note` | The step whose `failed` decision ended the run. |
| `by`, `at` | Every completed step: who and when. A wait records `by: "server"`; a `wait_for` records the sender and the event `data` as `output`. A finish and a parallel step record nothing. |
| `history` | Earlier completions of the same step, oldest first, each with the fields above. |

When an approval is rejected into `on_reject`, every step completed after the returned-to step, other than the rejecting approval itself, moves its current fields into `history` and has no current value until it completes again. The rejecting approval keeps `decision: rejected`, `note`, `by` and `at` current until it completes again, so the returned-to step reads the note at `steps.<id>.note`. An agent or person doing a step sees the whole run data. There are no input mappings. If a process grows large enough that this is a problem, it is two processes.

### 3.2 Moving on

At start, the first step in the list is ready. When a step completes, the server makes the step named by `next` ready. When `next` is a list, the submission or `completed` decision MUST include `next: <one of them>` and a non-blank `reason`, and the server records both. Supplying them on a step whose `next` is not a list is a `not_accepted` issue; supplying them on any decision other than `completed` is `invalid`. An approval continues at `next` when approved and at `on_reject` when rejected; the rejection note is kept in `steps.<id>.note`.

A `finish` ends the run with its outcome.

### 3.3 Agent loop

```
get_work   → ready agent steps the caller may claim, with the exact claim call to make
claim      → a token, its expiry, the step, the body of PROCESS.md, the run data,
             and the handoff: what the previous attempt at this step left behind
… do the work with your own tools …
renew      → extend the lease; optionally leave a progress note for whoever comes next
upload     → store a file, get its id (only when evidence: [file] is required)
submit     → output, evidence, a summary of what you did, and the chosen next.
             Accepted, or not with the reasons
escalate   → "I cannot finish this": the step goes to a person with your note
```

A claim lasts the server's lease, shown in `describe` and returned as `expiresAt`. `renew` sets a new expiry of now plus the lease and returns a new token; the old token stops working at once. `renew` MAY carry `progress`, a short note of what is done so far; the server keeps the latest one on the step while it is claimed.

**Handoff.** Every `claim`, every `get_work` item and every work item carries `handoff`: what the previous attempt at this step left behind, or `null` on a first attempt. It holds the last `progress` note if a lease expired, the `note` if a person returned the step, the `issues` if the last submission was not accepted, and the `check` result if a check sent it back (check profile). A new agent reads one field and knows the state of play. When a lease ends the step is claimable immediately. A lost `claim` or `submit` response is retried with the same `requestId` and returns the original result, token included, even if the lease has ended since; the agent reads `expiresAt`.

An agent that escalates loses its claim. The step waits on a person in the organization's operator queue, or wherever the server is configured to send escalations.

In a run started with `mode: test`, every `get_work` item, claim and work item says so. Agents MUST NOT cause real external effects, the server MUST NOT send real notifications, and people are told the run is a test.

A submission carries `summary`: a few sentences on what was done, what was found, and what was left undone, in the submitter's words. It is recorded beside the output, read by approvers, and kept in `history` when the step runs again. It is required from agents and optional from people.

### 3.4 People

A person step creates one **work item**. A role maps to a group at import; the item is shared, any member may complete it, and the first decision wins. `initiator` is the person who started the run; a run started by an agent has no initiator, and a step assigned to `initiator` is refused at start. A `person` path that resolves to nothing fails the run.

A person finds their work the same way an agent does: `get_work` called by a person lists the open work items assigned to them, oldest first, each with the exact `decide` call to make and the decisions allowed on it. A work item carries the step's `task` or `approve` text, its `output` and `evidence` requirements and allowed `next` list exactly as `claim` gives them to an agent, a `snapshot`, `due`, `handoff`, and that `decide` call. The body and the run data come from `get_run`. A person completes it with `decide`:

| Step | Decision | Effect |
|---|---|---|
| task | `completed` with `output`, `evidence`, and `next` when it is a list | Same acceptance as an agent submission. |
| approve | `approved` | Continue at `next`. |
| approve | `rejected` with `note` | Continue at `on_reject`, or end the run `rejected`. |
| escalated agent step | `returned` with `note` | The step is ready again; the note travels in `handoff.returned` to the next claim or, for a task, to the assignee's work item. The `due` clock does not restart; it is the same attempt. |
| escalated agent step | `completed` with `output` and `evidence` | The person finishes it in the agent's place, under the same acceptance. |
| any | `failed` with `note` | The run ends `failed`. |

The `snapshot` is a hash of the run data at the time of the read that returned it; every `get_run` returns a current one. Every decision carries it. If run data changed since that read, the decision is refused as `stale` and the person reads again. A decision on an item that was already decided is refused as `conflict` with `already_decided` before the snapshot is checked, so a second group member learns the truth rather than `stale`.

### 3.5 States

Run: `active`, `waiting` (only people, timers, events or escalations are pending), `ended` with `outcome` (a finish outcome, `rejected`, or `failed`), `cancelled`.

Step: `ready`, `claimed`, `waiting`, `done`, `failed` (its `failed` decision ended the run), `cancelled`, or `null` when the step is not currently reached; a rewound step is `null` and keeps its `history`.

An ended or cancelled run records `ended: { by, at, note }`: the cancel reason, the `failed` note, the rejection note when an approval without `on_reject` ends it, or `by: "server"` with a null note for a finish. A `person` path that resolves to nothing, or a `wait_until` value that is missing or not a datetime, fails that step with `by: "server"` and a note, and so ends the run.

When a run ends or is cancelled, every unfinished step becomes `cancelled`, every open work item is removed, and every outstanding token stops working. From then on every write to the run other than a replay returns `conflict` with `issues: ["run_ended"]`, checked before anything else. An operator can `cancel` an `active` or `waiting` run with a reason; cancelling an ended run is a `conflict`. Cancelling does not undo anything the run caused outside the server.

### 3.6 Events

`send_event { runId, name, data?, requestId }` delivers an event to one run. It is accepted from an agent identity or from an operator. A server holds events per run: an event that arrives before its `wait_for` step is ready is kept and consumed when the step becomes ready; each delivery satisfies one wait. Held events with the same name are consumed oldest first. An event with no matching wait, now or later, is held until the run ends and then discarded. `data` becomes `steps.<id>.output` of the wait step. Names match the step id pattern and compare exactly.

## 4. MCP tools

All tools are MCP tools over Streamable HTTP with OAuth 2.1 bearer tokens. Every result is `{ ok: true, data }` or `{ ok: false, error: { code, message, issues? } }`. Exact arguments and results, with one example each, are in [tools.md](tools.md); they are part of the specification.

A write returns only after every automatic transition it triggered has been applied: the run view it returns already shows the steps and work items the write made ready. Every write carries a client `requestId`, unique within the organization. Repeating it with the same arguments returns the original result and changes nothing. Repeating it with different arguments is a `conflict`. A call that failed is not recorded, so a retry can succeed.

### 4.1 Required

| Tool | Who | Purpose |
|---|---|---|
| `describe` | anyone | Protocol version, profiles, lease seconds, `person` format. |
| `list_processes` | anyone | Published processes: name, description, version number, `inputs`. |
| `start_run` | agent, person | Inputs are checked against `inputs`. Returns the run view. |
| `get_run` | anyone in the organization | The run view: run data, step states, open work items. Never a token. |
| `get_work` | agent, person | For an agent: ready agent steps it may claim, each with the exact `claim` call. For a person: their open work items, each with the exact `decide` call. Oldest first, paged. |
| `claim` | agent | Token, expiry, step, body, run data. |
| `submit` | agent | Output, evidence, chosen next. |
| `escalate` | agent | Hand the step to a person with a note. |
| `decide` | person | §3.4. A server MAY also offer a UI; it MUST apply the same rules. |
| `cancel` | operator | §3.5. |

### 4.2 Optional

| Tool | Who | Purpose |
|---|---|---|
| `renew` | agent | New token and expiry. |
| `upload` | agent, person | Store a file, get `{ id, sha256 }`. Required when any published process uses `evidence: [file]`. |
| `send_event` | agent, system | §3.6. Required when any published process uses `wait_for`. |
| `list_runs` | anyone in the organization | Filter by process, state, and any input field. |

A server MUST refuse to publish a process that needs a tool it does not offer.

### 4.3 Errors

| Code | Meaning | Agent does |
|---|---|---|
| `invalid` | Bad arguments. | Fix the call. |
| `not_accepted` | Submission broke a rule; `issues` lists each one. Nothing changed. | Fix the work, submit again. |
| `stale` | The token, claim or snapshot is no longer current. A token that does not verify at all is `invalid`. | Stop. Call `get_work` or `get_run` again. |
| `conflict` | Someone else holds it, it is already decided, or the `requestId` was reused with other arguments. | Move on. |
| `forbidden` | Not allowed for this identity. | Stop. |
| `not_found` | No such run, step or process in this organization. | Stop. |
| `retry` | Transient server state. | Retry with the same `requestId`. |

## 5. Evidence

An evidence item is `{ kind, ref }`.

| Kind | `ref` | Server verifies |
|---|---|---|
| `file` | An `upload` id | Exists in this organization; the stored bytes still match the hash recorded at upload. |
| `link` | A URL or document number | Nothing. Recorded as stated. |

`evidence: [file]` is met by at least one `file` item; more items of any kind may be attached. Free-text links never stand in for verified evidence. Wherever run data is returned, each `file` item also carries a short-lived `url` so agents and people can open it. Uploaded files are kept as long as the run record.

## 6. What a server guarantees

1. **Order.** Work exists only along the declared steps. A step becomes ready only at start, when its predecessor completed into it, when an approval rejected into it, or when a wait timed out into it.
2. **One actor at a time.** A claimed step belongs to one caller until it is submitted, escalated, or its lease ends. A submission with a token from a lease that ended, or from a claim that was replaced, is refused as `stale`. Tokens are opaque, unforgeable, and never appear in `get_run`.
3. **Acceptance.** A submission completes a step only when every non-optional `output` field is present with the declared type and within `one_of` (including inside lists), no undeclared field is present, every listed evidence kind is attached and verified, a chosen `next` is one of the allowed ones with a reason, and the run data stays within the server's `limits.runDataBytes`. The same cap bounds run inputs at start and event data when held. Otherwise nothing changes and the caller gets every issue at once. A call whose arguments break the tool's schema, such as a missing required argument, is `invalid` before any of this.
4. **People decide.** Only a human identity completes a task, approval or escalation. `decide` from an agent identity is always refused. Whether a person's credential is being used by that person is established by the server's authentication, outside this protocol; a server MUST NOT accept a person's credential presented by an agent client unless its authentication establishes that. A decision whose `snapshot` is not current is refused.
5. **Immutability.** Publishing creates a numbered version with a content hash: SHA-256 over the RFC 8785 canonical JSON of the parsed frontmatter, one newline character, then the body text exactly as written. A published version never changes. A run pins its version for life.

Plus one housekeeping rule: every write is idempotent on `requestId`, and the server, not the client, resolves concurrent writes to one run.

A server **conforms** when it keeps these five rules, offers the required tools with the contracts in the tools document, refuses what §2 says to refuse, and reports what it offers in `describe`.

## 7. Packages

A process travels as its folder: `PROCESS.md` alone in core, plus whatever a profile adds. On import a server creates a draft, asks the importer to map every role name in `person:` to a group, and publishes only on request. A package grants nothing in the organization that imports it.

## 8. Profiles (not core)

Each is a separate short document. A process uses a profile by using its keys; there is no declaration field. A server advertises what it implements in `describe`, and refuses at publication a process whose keys need a profile it lacks. The [independent reviews](../research/README.md) found a demonstrated need for `parallel`; `check` is included because it is how a process earns back the approvals it starts with.

| Profile | Adds | Use when |
|---|---|---|
| `parallel` | A `parallel:` step that makes several branches ready at once and continues when all have joined. Specified in [profiles/parallel.md](profiles/parallel.md). | Several people or agents must work at the same time. |
| `check` | A `check:` key on agent and task steps: a model answers yes/no, choice or rubric questions about the submission after the objective rules; fail sends it back, unsure sends it to a person. Specified in [profiles/check.md](profiles/check.md). | Approvals should happen only when a model is unsure. |

These two are the only profiles. Others, such as per-item fan-out, deterministic routing tables, subprocesses or server-executed actions, are defined only when a written process cannot be expressed without them, and each would be one page in this style.
