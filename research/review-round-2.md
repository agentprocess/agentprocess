Sources: `agentprocess-core-for-review.md` and `agentprocess-tools-for-review.md`.

I cannot verify which model handled round one or change this chat’s model. This review uses only these two documents.

Your prompt has no Task 2 heading. I interpret “give the two behaviors” as asking for concrete competing behaviors at specification gaps. I distinguish those from cases where the contracts permit only one behavior.

## Reading the walkthroughs

These are hypothetical conformance traces, not results from a running implementation.

- Each call below has exact example arguments. Tokens, work-item IDs, file IDs and snapshots are hypothetical values previously returned by the server.
- After `→`, I give the exact **result fields read**, using projections rather than reproducing every field of the run view.
- Every call first reads `ok`. On failure, it reads `error.code`, `error.message`, and `error.issues` if present.
- For a successful response, `D` means its outer `data`. Thus `D.data.inputs` means run inputs, while `D.run.state` means run state.
- Every successful claim reads `D.token`, `D.expiresAt`, `D.mode`, `D.step` in full, `D.body`, and `D.data` in full.
- Every human inspection reads `D.run`, `D.body`, `D.data`, `D.steps`, and the relevant entry in `D.workItems`, including `id`, `stepId`, `kind`, `text`, `assignedTo`, `snapshot`, `due`, `escalation`, and `next`.
- Every successful submission reads `D.accepted`, `D.run`, `D.steps`, `D.data.steps`, and `D.workItems`. Every successful decision reads those same fields except `accepted`, which is not promised for `decide`.
- Human calls are made by an authenticated human in the assigned group. External business work occurs between protocol calls; these documents do not specify the tools used for that work.

Before **each** run, the agent can execute this same read-only discovery sequence:

```text
describe {}
→ D.protocol="agentprocess", D.version="core-2",
  D.profiles=[], D.leaseSeconds=600, D.person.format="email",
  D.tools
  Also read D.maxUploadBytes if supplied; otherwise use 10 MiB.

list_processes {}
→ D.items[].name, description, version, inputs; D.nextCursor=null
```

Assume the relevant process is published as version `1`, the required tools are advertised, and role names have been mapped at import. Process 10 instead assumes `D.profiles=["parallel"]`.

Discovery is sufficient to start a known published process. It does not establish that every later human work item is self-describing.

# Task 1 — Ten complete processes

## 1. Invoice approval

```markdown
---
name: invoice-approval
description: Validate a USD invoice and obtain approval at the required spending level.
inputs:
  invoiceRef: string
  amount: number
steps:
  - id: validate
    agent: |
      Read the invoice and purchasing record. Confirm that amount is the
      non-negative invoice total in USD and that the goods or services
      were received. Attach the invoice reference as link evidence.
      If the invoice is invalid, choose invalid_invoice.
      Otherwise choose manager when amount is at most 10000, finance
      when amount is above 10000 and at most 100000, or executive when
      amount is above 100000.
    output:
      summary: string
    evidence: [link]
    next: [manager, finance, executive, invalid_invoice]
  - id: manager
    person: budget-manager
    approve: Approve this invoice only if the purchase and amount are justified.
    due: 2d
    next: approved
  - id: finance
    person: finance-approver
    approve: Approve this invoice only after checking budget and purchasing compliance.
    due: 2d
    next: approved
  - id: executive
    person: chief-financial-officer
    approve: Approve this invoice only after checking the business case and funding.
    due: 2d
    next: approved
  - id: invalid_invoice
    finish: invalid
  - id: approved
    finish: approved
---

# Invoice approval

Accounts payable starts one run per invoice. Amounts are USD. An invoice
of exactly 10000 goes to the budget manager; exactly 100000 goes to finance.
Larger invoices go to the chief financial officer. Approval authorizes
payment but does not perform it. A rejection ends this request.
```

**Fit: fits cleanly within the stated trust model.**

The routing is expressible. It is not a server-enforced spending control. The core explicitly excludes “approval authority matrices” and says that finer validation “goes in the instructions.” I would not report missing deterministic thresholds as an accidental omission: the draft deliberately puts that in `rules`.

### Agent run

Use an invoice of USD 125,000.

```text
start_run {"process":"invoice-approval","version":1,"inputs":{"invoiceRef":"INV-410","amount":125000},"mode":"live","requestId":"p1-start"}
→ D.run.id="r1", D.run.state="active", D.steps[validate].state="ready"

get_work {"process":"invoice-approval","limit":20}
→ D.items entry for r1/validate, including claim.tool="claim",
  claim.arguments.runId="r1", claim.arguments.stepId="validate",
  mode="live", note=null; D.nextCursor=null

claim {"runId":"r1","stepId":"validate","requestId":"p1-claim"}
→ D.token="t1", D.step.id="validate",
  D.step.next=["manager","finance","executive","invalid_invoice"]

submit {"token":"t1","output":{"summary":"Invoice matches the purchase order and receipt; total USD 125000."},"evidence":[{"kind":"link","ref":"INV-410"}],"next":"executive","reason":"USD 125000 is above USD 100000.","requestId":"p1-submit"}
→ D.accepted=true, D.run.state="waiting",
  D.workItems[0].id="w1", stepId="executive", snapshot="s1"
```

The human sequence below follows. The agent then observes:

```text
get_run {"runId":"r1"}
→ D.run.state="ended", D.run.outcome="approved",
  D.data.steps.executive.decision="approved"
```

**First undefined point:** none on this successful protocol path.

### Approver run

```text
get_run {"runId":"r1"}
→ executive work item w1, snapshot s1; invoice inputs,
  validation summary and link evidence

decide {"runId":"r1","workItemId":"w1","snapshot":"s1","decision":"approved","requestId":"p1-approve"}
→ D.run.state="ended", D.run.outcome="approved",
  D.data.steps.executive.decision="approved"
```

A real CFO can work with this if the invoice reference is accessible. A real organization would refuse to treat it as an independently enforced authority matrix. That refusal concerns the stated product boundary, not an undefined contract.

### Failure: rejection

In an alternative run state before approval:

```text
decide {"runId":"r1","workItemId":"w1","snapshot":"s1","decision":"rejected","note":"The purchase was not authorized.","requestId":"p1-reject"}
→ D.run.state="ended", D.run.outcome="rejected",
  D.data.steps.executive.decision="rejected",
  D.data.steps.executive.note="The purchase was not authorized."
```

No undefined behavior. The core says the default rejection outcome is `rejected`.

**Two concrete behaviors:** if the agent incorrectly chooses `manager` with a nonblank reason, the server can accept it because it is an allowed edge. Rejecting it specifically because the amount exceeds 100,000 requires a business rule beyond core acceptance. That is a deliberate boundary, not two equally specified threshold implementations.

---

## 2. Employee onboarding

```markdown
---
name: employee-onboarding
description: Complete HR, IT and hiring-manager onboarding for a new employee.
inputs:
  employeeRef: string
  manager: person
  startAt: datetime
  startDate: date
steps:
  - id: hr
    person: hr-team
    task: |
      Complete the employment record and required checks in the HR system.
      Keep identity documents there. Submit the employee record reference
      and confirm the checks are complete.
    output:
      recordRef: string
      checksComplete: boolean
    evidence: [link]
    next: it
  - id: it
    person: it-team
    task: |
      Prepare the employee's account, required access and equipment.
      Use the HR record to identify the employee. Do not put passwords
      in this run. Submit the service ticket reference.
    output:
      ticketRef: string
    evidence: [link]
    next: first_day
  - id: first_day
    wait_until: inputs.startAt
    next: manager
  - id: manager
    person: inputs.manager
    task: |
      Welcome the employee, review responsibilities and complete the
      first-day checklist. Submit the checklist reference.
    output:
      checklistRef: string
    evidence: [link]
    next: done
  - id: done
    finish: onboarded
---

# Employee onboarding

HR starts this once the hire is confirmed. The start date is the employee's
local calendar date; startAt is the agreed first-day meeting time with an
offset. HR and IT prepare in sequence. The hiring manager completes the
first-day checklist. Sensitive records stay in the systems that own them.
```

**Fit: fits cleanly as a sequential onboarding process.**

The requirement does not say HR and IT must run simultaneously. Serializing them is acceptable here; it would not be acceptable for process 10.

### Agent run and human handoffs

```text
start_run {"process":"employee-onboarding","version":1,"inputs":{"employeeRef":"EMP-27","manager":"manager@example.com","startAt":"2026-10-07T09:00:00+05:30","startDate":"2026-10-07"},"mode":"live","requestId":"p2-start"}
→ D.run.id="r2", D.run.state="waiting",
  D.workItems[0].id="w2h", stepId="hr", snapshot="s2h"

get_run {"runId":"r2"}
→ HR work item and run data
```

There is no agent work to claim. The HR, IT and manager calls below advance the run. Afterwards:

```text
get_run {"runId":"r2"}
→ D.run.state="ended", D.run.outcome="onboarded"
```

### Human run through `get_run` and `decide`

```text
get_run {"runId":"r2"}
→ HR work item w2h, snapshot s2h, text and body
```

**First undefined point for a document-naive human client:** the returned work item does not contain `output` or `evidence` requirements.

The task says “submit the employee record reference,” but the client cannot derive the exact keys `recordRef` and `checksComplete` or the mandatory evidence shape. The tools document gives `text`, `next`, and assignment fields; it does not give the task schema.

Using the PROCESS.md knowledge supplied to this reviewer, the continuation is:

```text
decide {"runId":"r2","workItemId":"w2h","snapshot":"s2h","decision":"completed","output":{"recordRef":"HR-27","checksComplete":true},"evidence":[{"kind":"link","ref":"HR-27"}],"requestId":"p2-hr"}
→ IT work item w2i, snapshot s2i

get_run {"runId":"r2"}
→ IT work item w2i, snapshot s2i

decide {"runId":"r2","workItemId":"w2i","snapshot":"s2i","decision":"completed","output":{"ticketRef":"IT-27"},"evidence":[{"kind":"link","ref":"IT-27"}],"requestId":"p2-it"}
→ D.run.state="waiting", D.steps[first_day].state="waiting"
```

After the specified time:

```text
get_run {"runId":"r2"}
→ manager work item w2m, snapshot s2m, assignedTo identifying the manager

decide {"runId":"r2","workItemId":"w2m","snapshot":"s2m","decision":"completed","output":{"checklistRef":"ONBOARD-27"},"evidence":[{"kind":"link","ref":"ONBOARD-27"}],"requestId":"p2-manager"}
→ D.run.state="ended", D.run.outcome="onboarded"
```

A manager should refuse a client that asks them to guess JSON field names. A server-specific UI could hide this problem, but the portable contract cannot.

**Smallest repair:** include `output` and `evidence` on task and escalated-agent work items, just as `claim.step` already does.

### Failure: HR cannot complete checks

```text
get_run {"runId":"r2"}
→ HR work item w2h, snapshot s2h

decide {"runId":"r2","workItemId":"w2h","snapshot":"s2h","decision":"failed","note":"Required employment verification failed.","requestId":"p2-fail"}
→ D.run.state="ended", D.run.outcome="failed"
```

No undefined behavior for failing the run.

**Two behaviors at the schema gap:** one client guesses `employeeRecord` and receives `not_accepted`; another knows to send `recordRef` and succeeds. The work-item result supplies no basis for choosing between those keys.

---

## 3. Customer support escalation

```markdown
---
name: support-escalation
description: Resolve a customer issue at tier 1 or hand it to engineering.
inputs:
  ticketRef: string
steps:
  - id: triage
    agent: |
      Read the support ticket and perform documented tier-1 troubleshooting.
      If resolved, choose respond. Otherwise choose engineering and
      describe the reproducible problem, customer impact and steps tried.
    output:
      summary: string
    next: [respond, engineering]
  - id: engineering
    person: engineering-support
    task: |
      Investigate the issue and record a fix or supported workaround in
      the ticket. Submit the customer-safe resolution summary and attach
      the engineering ticket reference.
    output:
      resolution: string
    evidence: [link]
    due: 1d
    next: respond
  - id: respond
    agent: |
      Read the current triage and engineering results. Explain the
      resolution to the customer in the support ticket and close it.
      Use the ticket reference as the external operation's deduplication
      key where the support system supports one.
    output:
      replyRef: string
    evidence: [link]
    next: done
  - id: done
    finish: resolved
---

# Customer support escalation

Run this for one customer support ticket. Tier 1 may resolve it directly.
Otherwise engineering owns the investigation before the customer receives
a closing response. Keep technical secrets out of the customer reply.
```

**Fit: fits cleanly.**

A business escalation to engineering is a declared task. It does not have to use the `escalate` tool, which transfers an agent’s unfinished step to the operator queue.

### Agent run

```text
start_run {"process":"support-escalation","version":1,"inputs":{"ticketRef":"SUP-81"},"mode":"live","requestId":"p3-start"}
→ D.run.id="r3", triage ready

get_work {"process":"support-escalation","limit":20}
→ r3/triage and exact claim template

claim {"runId":"r3","stepId":"triage","requestId":"p3-claim-triage"}
→ D.token="t3a", triage instructions, output schema and branch choices

submit {"token":"t3a","output":{"summary":"Export fails consistently after documented recovery steps."},"next":"engineering","reason":"Tier-1 troubleshooting did not resolve the reproducible failure.","requestId":"p3-submit-triage"}
→ engineering work item w3, snapshot s3
```

After the engineering sequence below:

```text
get_work {"process":"support-escalation","limit":20}
→ r3/respond and exact claim template

claim {"runId":"r3","stepId":"respond","requestId":"p3-claim-respond"}
→ D.token="t3b", D.data.steps.engineering.output.resolution

submit {"token":"t3b","output":{"replyRef":"SUP-81-REPLY"},"evidence":[{"kind":"link","ref":"SUP-81-REPLY"}],"requestId":"p3-submit-respond"}
→ D.accepted=true, D.run.state="ended", D.run.outcome="resolved"
```

The agent-facing happy path is defined.

### Engineering reviewer run

There is no approval step; engineering is the human worker.

```text
get_run {"runId":"r3"}
→ engineering work item w3, snapshot s3, triage summary

decide {"runId":"r3","workItemId":"w3","snapshot":"s3","decision":"completed","output":{"resolution":"The export service was repaired; retry the export."},"evidence":[{"kind":"link","ref":"ENG-81"}],"requestId":"p3-engineering"}
→ respond ready
```

The first undefined client requirement is again at `get_run`: the work item does not expose the `resolution` output field or required link evidence. The exact call above requires prior PROCESS.md knowledge.

### Failure: the response agent cannot access the support system

At the claimed `respond` step:

```text
escalate {"token":"t3b","note":"Support-system access is denied; restore access or send the response manually.","requestId":"p3-escalate"}
→ respond waiting, work item w3e with snapshot s3e,
  escalation.note as sent, escalation.by and escalation.at
```

Operator:

```text
get_run {"runId":"r3"}
→ w3e, snapshot s3e, escalation details

decide {"runId":"r3","workItemId":"w3e","snapshot":"s3e","decision":"returned","note":"Access restored. Check for an existing reply before sending.","requestId":"p3-return"}
→ respond ready, D.data.steps.respond.note contains the return note
```

Agent:

```text
get_work {"process":"support-escalation","limit":20}
→ r3/respond, note="Access restored. Check for an existing reply before sending."

claim {"runId":"r3","stepId":"respond","requestId":"p3-reclaim"}
→ D.token="t3c", D.step.note contains the return note

submit {"token":"t3c","output":{"replyRef":"SUP-81-REPLY"},"evidence":[{"kind":"link","ref":"SUP-81-REPLY"}],"requestId":"p3-recovered"}
→ ended, outcome resolved
```

No undefined escalation/return transition. The contract expressly settles it.

**Two behaviors:** `returned` makes the step claimable again and exposes the note; retaining the old claim would violate the contract. For manual `completed` instead, the missing work-item schema reappears: the operator cannot discover `replyRef` from the result.

---

## 4. Contract review before signature

```markdown
---
name: contract-review
description: Obtain legal and finance approval of a contract before signature.
inputs:
  contractRef: string
steps:
  - id: prepare
    agent: |
      Retrieve the proposed contract, identify its exact revision and
      attach that revision as file evidence. Summarize obligations,
      commercial terms and changes requested in prior rejection notes.
      Do not alter an already approved revision without restarting review.
    output:
      revision: string
      summary: string
    evidence: [file]
    next: legal
  - id: legal
    person: legal-team
    approve: Approve the attached contract revision for legal acceptability.
    due: 2d
    on_reject: prepare
    next: finance
  - id: finance
    person: finance-team
    approve: Approve the same attached revision for financial acceptability.
    due: 2d
    on_reject: prepare
    next: signature
  - id: signature
    person: authorized-signers
    task: |
      Sign only the exact revision approved by both legal and finance.
      Record the executed agreement in the contract system and submit
      its reference. If it differs from the approved revision, fail this
      run and begin a new review.
    output:
      executedRef: string
    evidence: [link]
    next: done
  - id: done
    finish: signed
---

# Contract review

The contract owner starts this before signature. Legal and finance review
the same stored revision. A rejection returns to preparation and requires
both reviews again. Signature occurs in the organization's signing system.
```

**Fit: fits cleanly within the stated trust model.**

Sequential legal and finance reviews satisfy “legal and finance before signature.” No parallel addition is necessary.

### Agent run

For brevity, the illustrative contract file consists of the bytes `contract`.

```text
start_run {"process":"contract-review","version":1,"inputs":{"contractRef":"CON-19"},"mode":"live","requestId":"p4-start"}
→ D.run.id="r4", prepare ready

get_work {"process":"contract-review","limit":20}
→ r4/prepare

claim {"runId":"r4","stepId":"prepare","requestId":"p4-claim"}
→ D.token="t4", preparation instructions and file requirement

upload {"fileName":"contract.txt","contentType":"text/plain","contentBase64":"Y29udHJhY3Q=","requestId":"p4-upload"}
→ D.id="f4", D.sha256, D.bytes=8

submit {"token":"t4","output":{"revision":"rev-1","summary":"One-year agreement at the reviewed price."},"evidence":[{"kind":"file","ref":"f4"}],"requestId":"p4-submit"}
→ legal work item w4l, snapshot s4l;
  D.data.steps.prepare.evidence[0].sha256 and .url
```

After the human sequence:

```text
get_run {"runId":"r4"}
→ ended, outcome signed, signature.output.executedRef="SIGNED-19"
```

### Approver and signer run

```text
get_run {"runId":"r4"}
→ legal work item w4l/s4l, revision rev-1, attached file URL

decide {"runId":"r4","workItemId":"w4l","snapshot":"s4l","decision":"approved","requestId":"p4-legal"}
→ finance work item w4f/s4f

get_run {"runId":"r4"}
→ finance work item w4f/s4f, same revision and legal approval

decide {"runId":"r4","workItemId":"w4f","snapshot":"s4f","decision":"approved","requestId":"p4-finance"}
→ signature work item w4s/s4s

get_run {"runId":"r4"}
→ signature work item w4s/s4s, both approvals

decide {"runId":"r4","workItemId":"w4s","snapshot":"s4s","decision":"completed","output":{"executedRef":"SIGNED-19"},"evidence":[{"kind":"link","ref":"SIGNED-19"}],"requestId":"p4-sign"}
→ ended, outcome signed
```

Legal and finance can make their decisions from the result. The signer again lacks the machine-readable submission schema.

A real approver should not approve a mutable document reference without checking its revision. This process avoids that by using stored file evidence. The specification already supplies the necessary mechanism.

### Failure: finance rejects after legal approves

At `w4f/s4f`:

```text
decide {"runId":"r4","workItemId":"w4f","snapshot":"s4f","decision":"rejected","note":"Change payment terms to net 60.","requestId":"p4-finance-reject"}
→ prepare ready;
  D.data.steps.legal.history contains the previous approval;
  D.data.steps.finance.history contains the rejection and note;
  those downstream steps have no current completion

get_work {"process":"contract-review","limit":20}
→ r4/prepare

claim {"runId":"r4","stepId":"prepare","requestId":"p4-reclaim"}
→ D.token="t4r", D.data including finance rejection history
```

**First undefined point:** none in the rollback transition. The core explicitly says downstream completions move into `history`. The agent must inspect history, not assume the latest rejection remains in a current `note`.

**Two behaviors:** clearing the old legal approval is required; carrying it forward as current would violate §3.1. The schema gap at signature remains the same two-client problem identified in process 2.

---

## 5. Expense reimbursement

```markdown
---
name: expense-reimbursement
description: Review receipts, obtain manager approval and reimburse an employee.
inputs:
  expenseRef: string
  manager: person
  amount: number
steps:
  - id: check
    agent: |
      Read the expense claim and receipts from the expense system.
      Confirm that the USD amount matches the claim and that required
      receipts are present. Attach the receipt bundle as file evidence.
      Summarize exceptions for the manager.
    output:
      summary: string
    evidence: [file]
    next: manager
  - id: manager
    person: inputs.manager
    approve: Approve reimbursement of the stated amount after reviewing receipts and exceptions.
    due: 3d
    on_reject: check
    next: reimburse
  - id: reimburse
    agent: |
      Submit the approved reimbursement in the expense system.
      First check whether this expense reference has already been paid.
      Use the expense reference as the payment idempotency key where
      supported. If the payment outcome is uncertain, escalate instead
      of submitting a second payment.
    output:
      paymentRef: string
    evidence: [link]
    next: done
  - id: done
    finish: reimbursed
---

# Expense reimbursement

The expense reference identifies one claim. Bank details remain in the
expense system. A manager rejection returns the claim for correction.
The run ends only after a reimbursement reference is recorded.
```

**Fit: fits cleanly.**

Receipts, approval and payment confirmation all fit. Protocol idempotency does not make the external payment exactly once; the instructions handle uncertainty without requesting a new profile.

### Agent run

Illustrative receipt file bytes: `receipt`.

```text
start_run {"process":"expense-reimbursement","version":1,"inputs":{"expenseRef":"EXP-55","manager":"manager@example.com","amount":85.5},"mode":"live","requestId":"p5-start"}
→ D.run.id="r5", check ready

get_work {"process":"expense-reimbursement","limit":20}
→ r5/check

claim {"runId":"r5","stepId":"check","requestId":"p5-claim-check"}
→ D.token="t5a"

upload {"fileName":"receipt.txt","contentType":"text/plain","contentBase64":"cmVjZWlwdA==","requestId":"p5-upload"}
→ D.id="f5", D.sha256, D.bytes=7

submit {"token":"t5a","output":{"summary":"Receipts total USD 85.50; no exceptions."},"evidence":[{"kind":"file","ref":"f5"}],"requestId":"p5-submit-check"}
→ manager work item w5/s5
```

After approval:

```text
get_work {"process":"expense-reimbursement","limit":20}
→ r5/reimburse

claim {"runId":"r5","stepId":"reimburse","requestId":"p5-claim-pay"}
→ D.token="t5b"

submit {"token":"t5b","output":{"paymentRef":"PAY-55"},"evidence":[{"kind":"link","ref":"PAY-55"}],"requestId":"p5-submit-pay"}
→ D.accepted=true, ended, outcome reimbursed
```

No undefined point in the ordinary successful path.

### Approver run

```text
get_run {"runId":"r5"}
→ manager work item w5/s5, amount 85.5, summary, receipt URL and hash

decide {"runId":"r5","workItemId":"w5","snapshot":"s5","decision":"approved","requestId":"p5-approve"}
→ reimburse ready
```

This gives a manager the amount, evidence, and exact decision target. Nothing essential to this approval is missing.

### Failure: successful submit response lost, then replayed after lease expiry

Assume the server accepted `p5-submit-pay`, but its response did not reach the agent. After the original lease expiry:

```text
submit {"token":"t5b","output":{"paymentRef":"PAY-55"},"evidence":[{"kind":"link","ref":"PAY-55"}],"requestId":"p5-submit-pay"}
```

**First undefined point: the replay result is contradictory across the supplied rules.**

Core §3.3 says:

> “A lost `claim` or `submit` response is retried with the same `requestId` and returns the original result, token included; if the lease has ended since, `stale`.”

Core §4 says:

> “Repeating it with the same arguments returns the original result and changes nothing.”

The tools contract specifically gives the expiry override for `claim`, but does not explicitly give that replay override for `submit`.

**Two behaviors:**

1. Return `{ok:false,error:{code:"stale",message:"Lease expired"}}` under §3.3.
2. Return the original successful run view with `accepted:true` under the universal idempotency rule.

Both imply no second payment or second transition. They disagree about the response the client must handle.

The client can recover safely:

```text
get_run {"runId":"r5"}
→ ended, outcome reimbursed, reimburse.output.paymentRef="PAY-55"
```

**Smallest repair:** explicitly distinguish replay of a previously accepted submission from a first submission made with an expired token. No new tool is needed.

---

## 6. RFP with three suppliers

```markdown
---
name: three-supplier-rfp
description: Collect three comparable supplier quotes and select one supplier.
inputs:
  rfpRef: string
  suppliers:
    type: list
    items: string
steps:
  - id: quotes
    agent: |
      Confirm that suppliers contains exactly three distinct suppliers.
      If it does not, escalate for correction rather than inventing one.
      Obtain a current written quote from each for the same scope.
      Submit quotes as three objects containing supplier, quoteRef,
      amount and currency. Attach all three quote references as links.
      Summarize differences in scope, price and delivery.
    output:
      quotes:
        type: list
        items: object
      comparison: string
    evidence: [link]
    next: selection
  - id: selection
    person: procurement-team
    task: |
      Review all three quotes and select one of the quoted suppliers.
      Submit the selected supplier and explain the commercial choice.
      If fewer than three comparable quotes are present, fail the run.
    output:
      selectedSupplier: string
      rationale: string
    next: done
  - id: done
    finish: selected
---

# Three-supplier RFP

Procurement starts this with three named suppliers and one scope of work.
Selection is a human decision. This process records the choice; it does
not issue a purchase order. Supplier count and quote comparability are
checked by the participants.
```

**Fit: fits cleanly.**

The three quotes do not require `each`. One agent can collect a fixed small set and one person can choose. The schema does not enforce exactly three or the internal object fields, but the draft explicitly delegates finer checks to instructions.

### Agent run

```text
start_run {"process":"three-supplier-rfp","version":1,"inputs":{"rfpRef":"RFP-9","suppliers":["Acme","Birch","Cedar"]},"mode":"live","requestId":"p6-start"}
→ D.run.id="r6", quotes ready

get_work {"process":"three-supplier-rfp","limit":20}
→ r6/quotes

claim {"runId":"r6","stepId":"quotes","requestId":"p6-claim"}
→ D.token="t6a", D.expiresAt

renew {"token":"t6a","requestId":"p6-renew"}
→ D.token="t6b", D.expiresAt updated

submit {"token":"t6b","output":{"quotes":[{"supplier":"Acme","quoteRef":"Q-A","amount":9000,"currency":"USD"},{"supplier":"Birch","quoteRef":"Q-B","amount":8500,"currency":"USD"},{"supplier":"Cedar","quoteRef":"Q-C","amount":9200,"currency":"USD"}],"comparison":"Equivalent scope; Birch is lowest priced and meets the delivery date."},"evidence":[{"kind":"link","ref":"Q-A"},{"kind":"link","ref":"Q-B"},{"kind":"link","ref":"Q-C"}],"requestId":"p6-submit"}
→ selection work item w6/s6
```

After selection:

```text
get_run {"runId":"r6"}
→ ended, outcome selected,
  selection.output.selectedSupplier="Birch"
```

No agent-contract gap on this path.

### Procurement run

```text
get_run {"runId":"r6"}
→ selection work item w6/s6 and all three quotes

decide {"runId":"r6","workItemId":"w6","snapshot":"s6","decision":"completed","output":{"selectedSupplier":"Birch","rationale":"Lowest price for equivalent scope and acceptable delivery."},"requestId":"p6-select"}
→ ended, outcome selected
```

First missing information: the work item does not expose `selectedSupplier` and `rationale` as required output fields.

The procurement owner should also understand that the server would accept a fourth supplier name if submitted: membership in the quote list is an instruction-level business check. That limitation is explicitly settled by the draft.

### Failure: lease expires during collection

Before a successful renewal or submission:

```text
submit {"token":"t6a","output":{"quotes":[],"comparison":"Quotes incomplete."},"evidence":[{"kind":"link","ref":"RFP-9"}],"requestId":"p6-expired-submit"}
→ ok=false, error.code="stale"
```

Then:

```text
get_work {"process":"three-supplier-rfp","limit":20}
→ r6/quotes, provided nobody else has claimed it

claim {"runId":"r6","stepId":"quotes","requestId":"p6-reclaim"}
→ new token "t6c", current run data and instructions
```

No undefined expiry behavior. Unsubmitted collection progress is not in run data; the agent must recover it from the RFP system.

**Two behaviors:** rejecting the first post-expiry submission as `stale` is required; accepting it because the work was completed before expiry is prohibited. The task-schema gap has the two-client behavior described earlier.

---

## 7. Blog post review and publication

```markdown
---
name: blog-publication
description: Draft a blog post, obtain editorial approval and publish at the scheduled time.
inputs:
  briefRef: string
  publishAt: datetime
steps:
  - id: draft
    agent: |
      Write the post from the brief. Store the draft in the content
      system, attach its reference and submit its revision identifier.
      Address earlier editorial rejection notes.
    output:
      draftRef: string
      revision: string
    evidence: [link]
    next: editor
  - id: editor
    person: editors
    approve: |
      Review the identified draft revision for publication.
      Approve to publish at the scheduled time, or choose hold to end
      this request without publication. Reject when revisions are needed.
    on_reject: draft
    next: [schedule, held]
  - id: schedule
    wait_until: inputs.publishAt
    next: publish
  - id: publish
    agent: |
      Publish exactly the revision approved by the editor.
      Check whether that revision is already published before writing.
      Submit the public URL.
    output:
      url: string
    evidence: [link]
    next: published
  - id: held
    finish: held
  - id: published
    finish: published
---

# Blog publication

Marketing supplies a brief and publication time. The editor approves a
specific revision. Choosing hold closes this request without publishing.
A rejection sends the draft back for revision and another editorial review.
```

**Fit: fits with a workaround.**

Exact workaround:

```yaml
approve: |
  Review the identified draft revision for publication.
  Approve to publish at the scheduled time, or choose hold to end
  this request without publication. Reject when revisions are needed.
next: [schedule, held]
```

The actual step ID is `held`; “choose hold” is the business wording. More fundamentally, an editor must send `decision:"approved"` to put the article on hold. A real editorial owner will find that misleading: the approval record says approved although publication was withheld.

No new feature is necessary. Make the editor step a `task` with a `publish/hold` output and branch choice, or remove the optional hold route. I included it because it exposes an existing approval-branch contract problem.

### Agent run

```text
start_run {"process":"blog-publication","version":1,"inputs":{"briefRef":"BRIEF-7","publishAt":"2026-10-07T10:00:00Z"},"mode":"live","requestId":"p7-start"}
→ D.run.id="r7", draft ready

get_work {"process":"blog-publication","limit":20}
→ r7/draft

claim {"runId":"r7","stepId":"draft","requestId":"p7-claim-draft"}
→ D.token="t7a"

submit {"token":"t7a","output":{"draftRef":"POST-7","revision":"rev-2"},"evidence":[{"kind":"link","ref":"POST-7-REV-2"}],"requestId":"p7-submit-draft"}
→ editor work item w7/s7,
  next=["schedule","held"]
```

After editorial approval and the scheduled time:

```text
get_work {"process":"blog-publication","limit":20}
→ r7/publish

claim {"runId":"r7","stepId":"publish","requestId":"p7-claim-publish"}
→ D.token="t7b", draft revision and editorial record

submit {"token":"t7b","output":{"url":"https://example.com/blog/post-7"},"evidence":[{"kind":"link","ref":"https://example.com/blog/post-7"}],"requestId":"p7-submit-publish"}
→ ended, outcome published
```

### Approver run

```text
get_run {"runId":"r7"}
→ w7/s7, next=["schedule","held"], draft revision and evidence

decide {"runId":"r7","workItemId":"w7","snapshot":"s7","decision":"approved","next":"schedule","reason":"The revision is approved for the scheduled publication.","requestId":"p7-editor"}
→ schedule waiting
```

**First underspecified result:** where does the client read the recorded approval routing reason?

Core §3.2 promises:

> “the server records both”

But §3.1 assigns `next` and `reason` to “a completed agent or task step,” and the tools’ approval run-data shape lists:

> “`decision`, `note`, `by`, `at`, `history` instead of `output`, `evidence`.”

It does not explicitly specify approval `next` and `reason`. A client should not have to infer fields absent from the normative approval shape.

**Smallest repair:** explicitly include approval `next` and `reason` when a branch was chosen.

### Failure: editor rejects the draft

```text
decide {"runId":"r7","workItemId":"w7","snapshot":"s7","decision":"rejected","note":"Remove the unsupported performance claim.","requestId":"p7-reject"}
```

**First conflicting requirement:** must a rejected approval with list-valued `next` also provide `next` and `reason`?

- Core §3.2 says the “submission or decision MUST include” them whenever `next` is a list.
- The `decide` table requires them for `approved`, but lists only `note` for `rejected`.

**Two behaviors:**

1. Accept the call above and return to `draft`, as the decision table implies.
2. Reject it with `not_accepted` for missing `next` and `reason`, applying the blanket rule.

The second behavior asks an editor to choose a forward branch that rejection will not use. A real editor should refuse that interaction.

**Smallest repair:** say that branch arguments apply only to successful completion or approval, not rejection or failure.

---

## 8. Incident postmortem with tracked action closure

```markdown
---
name: incident-postmortem
description: Review an incident and keep the process open until every corrective action is closed.
inputs:
  incidentRef: string
steps:
  - id: draft
    agent: |
      Prepare the timeline, impact, causes and corrective actions from
      the incident record. Create one action ticket per corrective
      action in the issue tracker, each with an owner and due date.
      Submit the report reference and the list of action ticket references.
    output:
      reportRef: string
      actionRefs:
        type: list
        items: string
    evidence: [link]
    next: close_actions
  - id: close_actions
    person: incident-owner
    task: |
      Track every action in the issue tracker until it is closed with
      evidence. Keep this task open while any action is incomplete.
      Submit all closed action references and attach the tracker report.
      Do not treat reassignment or a future commitment as closure.
    output:
      closedActionRefs:
        type: list
        items: string
    evidence: [link]
    due: 30d
    next: review
  - id: review
    person: reliability-reviewers
    approve: |
      Confirm that the postmortem is accurate and every listed action
      has closure evidence. Reject if any action remains incomplete.
    on_reject: close_actions
    next: done
  - id: done
    finish: closed
---

# Incident postmortem

Run this after service is restored. Individual action owners, due dates
and evidence live in the issue tracker. The incident owner keeps the
process open until all action tickets are closed. A reliability reviewer
checks the complete set before the postmortem is closed.
```

**Fit: fits with a workaround.**

Exact lines:

```text
Create one action ticket per corrective action in the issue tracker,
each with an owner and due date.

Track every action in the issue tracker until it is closed with
evidence. Keep this task open while any action is incomplete.
```

The process tracks aggregate closure. Individual action ownership, status and due dates live elsewhere.

A real incident owner would find it wrong if this were sold as per-action tracking inside the process server: the server has one open task and one due date. If the organization already uses an issue tracker, the workaround is practical.

No core addition is necessary for the stated requirement. Only a stronger requirement for native per-action work items would justify `each`.

### Agent run

```text
start_run {"process":"incident-postmortem","version":1,"inputs":{"incidentRef":"INC-88"},"mode":"live","requestId":"p8-start"}
→ D.run.id="r8", draft ready

get_work {"process":"incident-postmortem","limit":20}
→ r8/draft

claim {"runId":"r8","stepId":"draft","requestId":"p8-claim"}
→ D.token="t8"

submit {"token":"t8","output":{"reportRef":"PM-88","actionRefs":["ACT-1","ACT-2"]},"evidence":[{"kind":"link","ref":"PM-88"}],"requestId":"p8-submit"}
→ close_actions work item w8c/s8c, due timestamp and overdue=false
```

After the human sequence:

```text
get_run {"runId":"r8"}
→ ended, outcome closed
```

### Incident owner and approver run

```text
get_run {"runId":"r8"}
→ w8c/s8c, actionRefs=["ACT-1","ACT-2"]

decide {"runId":"r8","workItemId":"w8c","snapshot":"s8c","decision":"completed","output":{"closedActionRefs":["ACT-1","ACT-2"]},"evidence":[{"kind":"link","ref":"TRACKER-88-CLOSED"}],"requestId":"p8-close-actions"}
→ review work item w8r/s8r

get_run {"runId":"r8"}
→ w8r/s8r, original action list, reported closure list and evidence

decide {"runId":"r8","workItemId":"w8r","snapshot":"s8r","decision":"approved","requestId":"p8-review"}
→ ended, outcome closed
```

The incident owner has the recurring task-schema problem; the final approver has enough fields for approval.

### Failure: reviewer finds an unclosed action

```text
decide {"runId":"r8","workItemId":"w8r","snapshot":"s8r","decision":"rejected","note":"ACT-2 has no verified closure evidence.","requestId":"p8-reject"}
→ close_actions waiting again with a new work item w8c2/s8c2;
  review rejection moved to history;
  close_actions due clock restarted

get_run {"runId":"r8"}
→ w8c2/s8c2, rejection history and previous closure submission
```

The returned-to step’s previous completion remains visible until replaced; §3.1 invalidates steps completed **after** it. Its step state and new work item indicate that fresh work is required. This is not unspecified.

**Two behaviors:** leaving the task open and notifying when overdue is required; automatically failing or advancing after 30 days would violate the `due` rule. Native per-action tracking versus external tracking is a scope choice, not an unresolved server behavior.

---

## 9. Customer offboarding

```markdown
---
name: customer-offboarding
description: Cancel services, settle the final invoice and delete eligible customer data.
inputs:
  customerRef: string
  deleteAt: datetime
  retentionNote:
    type: string
    optional: true
steps:
  - id: cancel_services
    agent: |
      Confirm the authorized offboarding request. Cancel customer services
      in the service system and prepare the final invoice. Check existing
      records before repeating either operation. Submit both references.
    output:
      cancellationRef: string
      invoiceRef: string
    evidence: [link]
    next: finance
  - id: finance
    person: finance-team
    approve: Confirm the final invoice and authorize proceeding to settlement.
    next: settlement
  - id: settlement
    wait_for: final_settlement
    timeout: 3d
    on_timeout: resolve_settlement
    next: retention
  - id: resolve_settlement
    person: accounts-receivable
    task: |
      Resolve settlement of the final invoice. Keep this task open until
      the invoice is settled or formally written off. Submit the
      settlement record and attach its reference. Fail the run if
      offboarding cannot proceed.
    output:
      settlementRef: string
    evidence: [link]
    next: retention
  - id: retention
    wait_until: inputs.deleteAt
    next: delete_data
  - id: delete_data
    agent: |
      Check the customer's retention and hold instructions in the system
      of record, including retentionNote when present. Delete only data
      eligible for deletion. Record retained data and its controlling
      reason in the deletion report. If a hold prevents completion,
      escalate rather than bypassing it. Submit the report reference.
    output:
      deletionReportRef: string
    evidence: [link]
    next: done
  - id: done
    finish: offboarded
---

# Customer offboarding

The approved deletion time comes from the organization's retention process.
The billing integration sends final_settlement only after recording
settlement of this run's final invoice. Service cancellation and deletion
occur in their systems of record and are not undone by cancelling this run.
```

**Fit: fits cleanly for the declared sequence.**

This does not promise atomic rollback of cancellation, invoicing and deletion. The core expressly says cancellation does not undo external effects.

### Agent run, with an early settlement event

```text
start_run {"process":"customer-offboarding","version":1,"inputs":{"customerRef":"CUS-9","deleteAt":"2026-11-06T00:00:00Z"},"mode":"live","requestId":"p9-start"}
→ D.run.id="r9", cancel_services ready

get_work {"process":"customer-offboarding","limit":20}
→ r9/cancel_services

claim {"runId":"r9","stepId":"cancel_services","requestId":"p9-claim-cancel"}
→ D.token="t9a"

submit {"token":"t9a","output":{"cancellationRef":"CANCEL-9","invoiceRef":"FINAL-9"},"evidence":[{"kind":"link","ref":"CANCEL-9"},{"kind":"link","ref":"FINAL-9"}],"requestId":"p9-submit-cancel"}
→ finance work item w9/s9
```

The billing system delivers before the wait is ready:

```text
send_event {"runId":"r9","name":"final_settlement","data":{"settlementRef":"SET-9"},"requestId":"p9-event"}
→ D.delivered=true, D.consumedBy=null
```

After finance approves, the held event satisfies `settlement`; the run waits at `retention`.

After `deleteAt`:

```text
get_work {"process":"customer-offboarding","limit":20}
→ r9/delete_data

claim {"runId":"r9","stepId":"delete_data","requestId":"p9-claim-delete"}
→ D.token="t9b",
  D.data.steps.settlement.output={"settlementRef":"SET-9"}

submit {"token":"t9b","output":{"deletionReportRef":"DELETE-9"},"evidence":[{"kind":"link","ref":"DELETE-9"}],"requestId":"p9-submit-delete"}
→ ended, outcome offboarded
```

A single early event is completely defined. Reporting it as a missing capability would be wrong.

### Approver run

```text
get_run {"runId":"r9"}
→ finance work item w9/s9, final invoice reference

decide {"runId":"r9","workItemId":"w9","snapshot":"s9","decision":"approved","requestId":"p9-finance"}
→ settlement done, retention waiting,
  D.data.steps.settlement.output={"settlementRef":"SET-9"}
```

The finance approval has sufficient information, assuming the invoice link is accessible.

### Failure: two events arrive before the wait

Before finance approves:

```text
send_event {"runId":"r9","name":"final_settlement","data":{"settlementRef":"SET-OLD"},"requestId":"p9-event-old"}
→ D.delivered=true, D.consumedBy=null

send_event {"runId":"r9","name":"final_settlement","data":{"settlementRef":"SET-CORRECTED"},"requestId":"p9-event-corrected"}
→ D.delivered=true, D.consumedBy=null

decide {"runId":"r9","workItemId":"w9","snapshot":"s9","decision":"approved","requestId":"p9-finance-after-events"}
→ settlement completes
```

**First undefined point:** which event populates `D.data.steps.settlement.output`?

The core says:

> “each delivery satisfies one wait”

It says nothing about ordering multiple held events with the same name.

**Two behaviors:**

1. FIFO: `output={"settlementRef":"SET-OLD"}`.
2. Consume the newest held matching event: `output={"settlementRef":"SET-CORRECTED"}`.

The single wait cannot distinguish these by contract. A business integration also needs to know that a correction does not necessarily replace an earlier delivery.

**Smallest repair:** define matching-event consumption order. FIFO is enough; replacement semantics are unnecessary unless explicitly wanted.

For the timeout path, the existing tools can finish `resolve_settlement`; its work item again lacks the output/evidence schema. No additional timeout tool is necessary.

---

## 10. Major incident response in parallel

This is an **assumed profile interpretation**, not a document that can be validated against the supplied specification. Only `branches` is introduced; its value and execution semantics are guesses.

```markdown
---
name: major-incident-response
description: Activate operations, security and communications together for a major incident.
inputs:
  incidentRef: string
  severity:
    type: string
    one_of: [sev1, sev2]
steps:
  - id: response
    agent: |
      Coordinate the active incident. Maintain the incident log, identify
      cross-team dependencies and keep the incident commander informed.
      Complete coordination only after the incident commander agrees
      that recovery is stable.
    branches: [operations, security, communications]
    next: recovered
  - id: operations
    person: operations-team
    task: |
      Begin immediately on activation. Restore service and record the
      recovery evidence in the incident system. Remain engaged until
      recovery is stable.
    output:
      recoveryRef: string
    evidence: [link]
    next: recovered
  - id: security
    person: security-team
    task: |
      Begin immediately on activation. Investigate security implications,
      contain any threat and record the security assessment. Confirm
      whether the recovery can proceed safely.
    output:
      assessmentRef: string
    evidence: [link]
    next: recovered
  - id: communications
    person: communications-team
    task: |
      Begin immediately on activation. Issue and maintain stakeholder
      updates. Record the final recovery communication.
    output:
      communicationRef: string
    evidence: [link]
    next: recovered
  - id: recovered
    finish: recovered
---

# Major incident response

Activation makes coordination, operations, security and communications
ready together. The incident is recovered only after coordination and
all three branches finish successfully. No branch may finish the run by
itself. A failed branch fails the run and cancels unfinished branches;
already performed external actions remain in effect.
```

**Fit: does not fit core. It fits only under invented parallel-profile semantics.**

The core says:

> “At start, the first step in the list is ready.”

And:

> “When a step completes, the server makes the step named by `next` ready.”

A sentence telling an agent to coordinate simultaneous work cannot create simultaneous server work items. This is the one process where instructions alone cannot satisfy the explicit requirement.

**Smallest addition:** a normative fork/join definition that activates these branches when the first step becomes ready, plus one specified failure policy. This need not require additional tools if existing run views and `get_work` results cover multiple active steps.

### Every profile question I had to answer myself

1. Is `branches` a list of step IDs, nested step lists, or a mapping?
2. Which step kinds may carry `branches`?
3. Does the fork occur when the containing step becomes ready, is claimed, or completes?
4. Does the containing step itself execute, or is it only a fork?
5. Are all branch entries activated atomically?
6. Does “at the same time” mean simultaneous readiness or simultaneous worker execution? Only the former is realistically under server control.
7. How is the join identified: shared `next`, containing-step `next`, or another declaration?
8. Does the containing step participate in the join?
9. Must all branches succeed, or can the join use another threshold?
10. Can one branch’s `next` reach the finish before siblings finish?
11. Are branch steps exempt from ordinary reachability analysis through `next`?
12. Does the acyclic graph rule include `branches` edges?
13. What happens if two branches point to the same intermediate step?
14. What happens when a branch fails, is rejected, escalates, or times out?
15. Does failure cancel sibling work and invalidate sibling claim tokens immediately?
16. Can failed branches be retried without repeating successful branches?
17. If rejection reopens part of a parallel run, which sibling results become history?
18. Do branch completions invalidate every other open work item’s run-wide snapshot?
19. If so, how does `get_run` provide a refreshed snapshot for an existing work item?
20. What state is the run in while one branch has agent work and others have human work?
21. Can branches be nested?
22. How does the process declare that it requires `parallel`, given that unknown frontmatter fields are rejected and no declaration field is defined?
23. How do versioning and conformance identify which parallel semantics are in force?

The body in this example answers some business questions but cannot make those answers enforceable profile semantics.

### Agent run

```text
start_run {"process":"major-incident-response","version":1,"inputs":{"incidentRef":"INC-100","severity":"sev1"},"mode":"live","requestId":"p10-start"}
```

**First undefined point:** the result of this call. The documents do not say whether the three branch work items exist yet—or whether this file is valid.

Under the assumptions written in the body, the expected projection would be:

```text
→ D.run.id="r10", response ready;
  operations work item w10o/s10o;
  security work item w10s/s10s;
  communications work item w10c/s10c
```

Conditional continuation:

```text
get_work {"process":"major-incident-response","limit":20}
→ r10/response

claim {"runId":"r10","stepId":"response","requestId":"p10-claim"}
→ D.token="t10", coordination instructions and run data
```

After the human branches complete:

```text
submit {"token":"t10","output":{},"requestId":"p10-submit"}
→ assumed ended, outcome recovered
```

This is an expected trace, not a trace justified by the one-line profile.

### Human branch runs

Under the assumed activation behavior:

```text
get_run {"runId":"r10"}
→ operations work item w10o/s10o

decide {"runId":"r10","workItemId":"w10o","snapshot":"s10o","decision":"completed","output":{"recoveryRef":"REC-100"},"evidence":[{"kind":"link","ref":"REC-100"}],"requestId":"p10-ops"}
→ assumed operations done; run not ended

get_run {"runId":"r10"}
→ security work item w10s with a current snapshot "s10s2"

decide {"runId":"r10","workItemId":"w10s","snapshot":"s10s2","decision":"completed","output":{"assessmentRef":"SEC-100"},"evidence":[{"kind":"link","ref":"SEC-100"}],"requestId":"p10-security"}
→ assumed security done; run not ended

get_run {"runId":"r10"}
→ communications work item w10c with a current snapshot "s10c2"

decide {"runId":"r10","workItemId":"w10c","snapshot":"s10c2","decision":"completed","output":{"communicationRef":"COMMS-100"},"evidence":[{"kind":"link","ref":"COMMS-100"}],"requestId":"p10-comms"}
→ assumed communications done; coordinator still active
```

These calls expose a second problem even after inventing fork/join semantics. Core §3.4 defines a snapshot as the hash when the work item **was created**, but says that after a stale decision “the person looks again.” The contract does not say whether looking again replaces the work item, refreshes its snapshot, or returns the same now-unusable creation snapshot.

A real incident responder will refuse an interface in which another team’s update permanently prevents submitting their work.

### Failure: sibling completion makes a decision stale

Suppose security loaded `s10s` before operations completed:

```text
decide {"runId":"r10","workItemId":"w10s","snapshot":"s10s","decision":"completed","output":{"assessmentRef":"SEC-100"},"evidence":[{"kind":"link","ref":"SEC-100"}],"requestId":"p10-security-stale"}
→ ok=false, error.code="stale"

get_run {"runId":"r10"}
```

**First undefined recovery behavior:** the snapshot on the still-open security work item.

**Two behaviors:**

1. Return a refreshed current snapshot, allowing resubmission.
2. Return the immutable creation snapshot, causing every subsequent submission to remain stale.

The prose defines the latter snapshot but appears to intend the former recovery. Specify the refresh mechanism.

For activation itself, two equally plausible interpretations of the one-line profile are also incompatible: fork when `response` becomes ready versus fork after `response` completes. The latter fails this process’s explicit activation requirement.

# Additional contract checks exposed by these processes

These are behavioral issues, not style suggestions.

## Agent-started runs with a later `initiator` assignment

An ordinary variation of onboarding is for a human starter to receive the final onboarding task:

```yaml
person: initiator
```

Core §3.4 says:

> “a run started by an agent has no initiator, and a step assigned to `initiator` is refused at start.”

The `start_run` contract narrows its explicit invalid case to:

> “a process whose first step is a person step assigned to `initiator`”

For a process with an agent first step and a later `initiator` task, the two behaviors are:

1. Reject `start_run` immediately, following the core.
2. Accept the run because the first step is not assigned to `initiator`, then reach an unassignable human step.

The second behavior cannot satisfy the broader core rule. The tool contract should state the same whole-process check, rather than giving a narrower condition that reads as the complete rule.

**Repair:** change “first step” to “any reachable step.” No feature addition.

## Human-presence assertion

The core requires a trusted human-presence assertion when an agent uses a person’s token. The `decide` contract contains no argument or transport description for it.

This does **not** make an ordinary authenticated human’s approval undefined. It does mean that an agent-assisted approval client cannot implement this requirement from these documents alone. One server may rely on its own browser session; another may require an additional assertion mechanism.

**Repair:** identify how the assertion is conveyed, or explicitly declare that agent-assisted human decisions require server-specific authentication outside this protocol.

## Some run-state results are not fully specified

The run can end `failed`, and steps can be `cancelled`, but the documents do not say what happens to every unfinished step when:

- a human sends `decision:"failed"`;
- an operator cancels a run;
- a branch failure ends a hypothetical parallel run.

The run-level result is clear. The remaining step-state and work-item projections are not completely stated.

Two possible views are an ended run retaining waiting step records versus an ended run marking all unfinished steps cancelled and removing open items.

**Repair:** one sentence defining terminal cleanup and claim invalidation. This matters to clients rendering work queues, not just presentation.

# Task 3 — Coverage and removal audit

“Used” below means exercised by an authored process, a walkthrough, or a concrete compatibility check above. A feature being unused by these ten cases is evidence to question it, not proof that it has no value.

## Format fields and kinds

| Field or kind | Used? | Remove? |
|---|---|---|
| `name` | All | No; identity and discovery. |
| `description` | All | No; selection before starting. |
| `inputs` | All | No. |
| `steps` | All | No. |
| Step `id` | All | No; routing and tool identity. |
| `agent` | All except onboarding | No. |
| `task` | 2, 3, 4, 6, 8, 9, 10 | No. |
| `approve` | 1, 4, 5, 7, 8, 9 | No. |
| `wait` | None | No; a relative delay is a small useful primitive. These examples happen to need calendar times instead. |
| `wait_until` | 2, 7, 9 | No. |
| `wait_for` | 9 | No. |
| `finish` | All | No. |
| `output` | All | No. Expose it in human work items. |
| `evidence` | All | No. |
| Scalar `next` | All | No. |
| List `next` | 1, 3, 7 | No. |
| Omitted `next` / following-step default | Not relied on | Keep; harmless author convenience. |
| `person` role name | All | No. |
| `person` data path | 2, 5 | No. |
| `person: initiator` | Compatibility check only | Keep; useful assignment shorthand. Fix start validation. |
| `on_reject` | 4, 5, 7, 8 | No. |
| Default rejected outcome | 1; also applies to 9 | No. |
| `due` | 1, 2 indirectly through work-item shape, 3, 4, 5, 8 | No; overdue work must remain visible. Process 2 itself declares no due interval. |
| `timeout` | 9 | No. |
| `on_timeout` | 9 | No. |
| Default `on_timeout = next` | Not exercised | Keep, but authors must understand that timeout can proceed without an event. |
| `branches` | 10, assumed only | Cannot count as implemented. Publish its semantics or omit the claim that this profile is available. |
| Markdown body | All | No; operational context. |

## Field schema

| Field/type/rule | Used? | Remove? |
|---|---|---|
| Bare type notation | All | No. |
| Object field notation with `type` | 2, 6, 8, 9, 10 | No. |
| Field `description` | None | Keep; useful in `list_processes.inputs` before a run exists. |
| `optional: true` | 9 | No. |
| `one_of` | 10 | No; also useful without parallel. |
| `items` | 6, 8 | No. |
| `items` as a type name | 6, 8 | No. |
| `items` as a field object | None | Keep if list-item constraints are intended; no need for another schema system. |
| `string` | All | No. |
| `number` | 1, 5 | No. |
| `boolean` | 2 | No. |
| `date` | 2 | Keep; business calendar dates differ from instants. |
| `datetime` | 2, 7, 9 | No. |
| `list` | 6, 8 | No. |
| `object` | 6 quote items | No. |
| `person` | 2, 5 | No. |
| Required unless optional | All | No. |
| Optional fields may be omitted | 9 | No. |
| `null` is not a field value | Not sent | Keep; avoids two absence representations. |
| Undeclared fields rejected | All submission contracts depend on it | No. |
| Values outside `one_of` rejected | 10 relies on it; no negative call | No. |
| Numbers not rounded | 5 | No. |
| Finer constraints in instructions | 1, 2, 6, 8, 9 | No additional schema requested. This is a deliberate trust boundary. |
| Server-defined person identity format | 2, 5 discovery and inputs | No. |
| `date` lexical format | 2 | No. |
| RFC 3339 offset requirement | 2, 7, 9 | No. |
| Duration integer + `m/h/d` | `d` used; `m/h` not used | Keep all three; no meaningful simplification from removing units. |

## Structural and execution rules

| Rule | Used? | Remove? |
|---|---|---|
| Name syntax and length | All names conform | No. |
| Name equals folder name | Assumed at publication; not exercised by run tools | Keep if folders are package identity. |
| Description length guidance | All | Keep; not an execution mechanism. |
| At least one step | All | No. |
| Exactly one kind per step | All | No. |
| Kind-specific optional-key restrictions | All | No. |
| YAML 1.2 core schema | Parsing all files relies on it | No. |
| Duplicate keys refused | No negative import test | Keep; ambiguous documents are dangerous. |
| Unknown fields refused | All; exception highlights profile 10 | No. |
| Step-ID syntax and uniqueness | All | No. |
| All edge targets exist | All | No. |
| Every step reachable | All core examples | No. Define reachability with `branches`. |
| Last non-finish needs `next` | No example ends with a non-finish | Keep; prevents accidental dead ends. |
| Finish has no `next` | All | No. |
| Forward graph acyclic | All core examples | Keep for this deliberately limited core. |
| Only `on_reject` goes backward | 4, 5, 7, 8 | No. |
| First step ready at start | All | No. |
| Scalar-next progression | All | No. |
| List-next choice and nonblank reason | 1, 3, 7 | No. Narrow its application to successful decisions. |
| Reject routing via `on_reject` | 4, 7, 8 | No. |
| Finish determines outcome | All | No. |
| Downstream invalidation on rejection | 4, 8 | No. |
| Earlier completions in history, oldest first | 4, 8 | No. |
| Whole run data visible to workers | All | Keep only with the stated policy of storing sensitive material elsewhere. |
| No input mappings | All | No new mapping system needed for these cases. |
| Body shown with claims and work items | All | No. |
| Catalog shows name and description | Discovery uses these and inputs | Keep, but “only name and description” must mean the human catalog display; `list_processes` expressly also returns version and inputs. |
| Role-to-group mapping at import | Assumed for all roles | No. |
| Shared group work item, first decision wins | Approvals depend on it; no competing successful decisions simulated | No. |
| Human-only completion | All human sequences | No. |
| Missing person-path value fails run | Relevant to 2, 5; not triggered | Keep. |
| Agent run has no initiator | Compatibility check | No. |
| One live claim per step | All agent runs | No. |
| Lease expiry releases claim | 6 | No. |
| Renew replaces token | 6 | No. |
| Expired/replaced token rejected | 6 | No. |
| Escalation invalidates token | 3 | No. |
| Escalation creates operator work item | 3 | No. |
| Returned escalation exposes note | 3 | No. |
| Human completes escalated step | Examined as alternate in 3 | Keep; add its schema to the work item. |
| `failed` ends the run | 2 | No. |
| Snapshot required | All decisions | No. |
| Stale snapshot refused | 10 | No. Specify recovery. |
| Due marks overdue, not failure | 8 | No. |
| Repeated step restarts due clock | 8 | No. |
| Past `wait_until` proceeds immediately | Not explicitly exercised | Keep; essential deterministic boundary case. |
| Invalid/missing `wait_until` fails | Not triggered | Keep. |
| Events held before wait readiness | 9 | No. |
| One delivery satisfies one wait | 9 | No. |
| Event data becomes wait output | 9 | No. |
| Event name syntax and exact match | 9 | No. |
| Timeout uses `on_timeout` or `next` | 9 format | No. |
| Cancel does not undo external effects | 9 | No. |
| Server serializes concurrent writes | Needed by shared work and 10 | No. |
| Writes idempotent by request ID | 5 | No. Resolve expiry precedence. |
| Reused ID with changed arguments conflicts | Not triggered | Keep. |
| Failed writes not recorded | Recovery reasoning in 10 | Keep. |
| Version pinned for run life | All starts pin version 1 | No. |
| Published version immutable | All | No. |
| Content hash construction | None of the walkthroughs can observe it | Keep immutability; the exact hashing rule needs separate interoperability validation. Do not remove it just because runtime tools do not expose it. |
| Import creates draft; explicit publish | Assumed, not exercised | Keep. These tools are runtime tools. |
| Package grants no organizational authority | Role assumptions | No. |
| Refuse publication when required tool absent | File and event processes rely on it | No. |
| Refuse unsupported profile | 10 | No. |
| Test mode suppresses real effects/notifications | No test-mode run | Keep; production agents need a safe rehearsal mode. |

The reference to a catalog showing “only `name` and `description`” is not sufficient evidence to demand removal of `inputs` from `list_processes`; that contract explicitly settles what discovery returns.

## Tools and arguments

| Tool | Used? | Remove? |
|---|---|---|
| `describe` | Shared discovery | No. |
| `list_processes` | Shared discovery | No. |
| `start_run` | All | No. |
| `get_run` | All | No. |
| `get_work` | All agent-work processes | No. |
| `claim` | All agent-work processes | No. |
| `submit` | All agent-work processes | No. |
| `escalate` | 3 | No. |
| `decide` | All | No. |
| `cancel` | No call | Keep for accidental or withdrawn runs. |
| `renew` | 6 | Keep optional. |
| `upload` | 4, 5 | No when file evidence is supported. |
| `send_event` | 9 | No when waits for events are supported. |
| `list_runs` | No call; all walkthroughs retain run IDs | Keep optional; useful for recovering runs and operational queues. |

| Argument family | Used? | Removal judgment |
|---|---|---|
| `requestId` | Every write | Keep. |
| `process` | Discovery selection, starts, work filter | Keep. |
| Explicit start `version` | All | Keep. |
| Latest-version default | Not used | Keep. |
| `inputs` | All starts | Keep. |
| `mode:"live"` | All starts | Keep `test` too. |
| `runId` | All | Keep. |
| `stepId` | Claims | Keep. |
| `token` | Agent writes | Keep. |
| `output` | Submissions and task decisions | Keep. |
| `evidence` | Submissions and task decisions | Keep. |
| `next` and `reason` | 1, 3, 7 | Keep. |
| `note` | Rejection, escalation, return, failure | Keep. |
| `workItemId`, `snapshot`, `decision` | All human sequences | Keep. |
| Upload filename, MIME type, base64 | 4, 5 | Keep. |
| Event `name`, `data` | 9 | Keep. |
| Event omitted `data` | Not used | Keep optionality; signal-only events are reasonable. |
| Pagination cursors | Read as null; continuation not exercised | Keep; company queues are not bounded by ten examples. |
| `get_work.limit` | Explicitly 20 | Keep bounded paging. |
| `get_work.process` | All work retrieval | Keep. |
| `list_runs` filters and pagination | Unused | Keep optional; no evidence for more query operators. |
| Cancel `reason` | Unused | Keep with cancellation. |

## Result fields

| Result fields | Used? | Remove? |
|---|---|---|
| `ok`, `error.code/message/issues` | All / negative traces | No. |
| `run.id/process/version/mode/state/outcome` | All | No. |
| `run.startedBy/startedAt/updatedAt` | Read in human run views; not decisive in examples | Keep for operational/audit context. |
| `data.inputs` | All | No. |
| `data.steps.*.output/evidence` | All | No. |
| `data.steps.*.next/reason` | Agent branches 1, 3 | No; explicitly extend approval representation. |
| `decision/note` | Human approvals/failures | No. |
| `by/at/history` | 4, 8 and audit inspection | No. |
| `steps[].id/kind/state` | All | No. |
| `steps[].due/overdue` | 8; other due-bearing steps | No. |
| Unreached step `state:null` | Branch alternatives | Keep; distinguish unreached from cancelled. |
| `workItems[].id/stepId/kind/text/assignedTo/snapshot/due` | Human paths | No. |
| `workItems[].escalation` | 3 | No. |
| `workItems[].next` | 7 | No. |
| Work-item test `mode` | Unused | Keep. |
| `body` | All | No. |
| Discovery protocol/version/profiles/lease/person/tools | All discovery | No. |
| `maxUploadBytes` | Default read in file workflows | Keep; explicit contract type would be useful. |
| Process catalog inputs/version | All starts | No. |
| Work-list mode/readySince/due/overdue/note | Work retrieval and 3 | No. |
| Work-list exact claim template | All claims | Keep; useful bootstrap information. |
| Claim token/expiry/step/schema/evidence/next/note | All claims | No. |
| Renew token/expiry | 6 | No. |
| Upload id/hash/bytes | 4, 5 | No. |
| Submit `accepted:true` | All submissions | **Candidate to remove.** `ok:true` already means submission accepted; no independent meaning is specified. |
| File evidence `sha256/url` | 4, 5 approvals | No. |
| Event `delivered/consumedBy` | 9 | Keep `consumedBy`; `delivered:true` is another redundant success flag, though less consequential. |
| `nextCursor` | Discovery/work retrieval | No. |
| `list_runs` summary fields | Unused | Keep with optional tool. |

## Error codes

| Code | Used? | Remove? |
|---|---|---|
| `invalid` | Initiator/start compatibility check | No; distinguishes malformed or invalid starts. |
| `not_accepted` | Human schema gap and branched rejection conflict | No. |
| `stale` | 5, 6, 10 | No. |
| `conflict` | Discussed in one-actor/first-decision rules; no call returning it | Keep; races happen in ordinary operation. |
| `forbidden` | Human-presence/assignment rules; no returned example | Keep; authorization failures must remain distinct. |
| `not_found` | No | Keep; stale links and nonexistent identifiers need a defined result. |
| `retry` | No | Keep if servers genuinely expose retryable conditions; otherwise implementations should not emit it speculatively. |

Specific conflict issue strings—`claimed_by_other`, `not_ready`, and `already_decided`—were not exercised as returned failures. Keep them: they explain materially different recovery situations.

The `not_accepted` examples for missing fields, wrong types, undeclared fields, missing evidence and missing branch arguments describe necessary acceptance failures. These ten processes do not need an expanded error taxonomy.

## Evidence and transport rules

| Rule | Used? | Remove? |
|---|---|---|
| Evidence `{kind,ref}` | All evidence-bearing processes | No. |
| File must exist in organization | 4, 5 rely on it | No. |
| File bytes verified against upload hash | 4, 5 rely on it | No. |
| Link is recorded without verification | Most examples | No; make no stronger claim about it. |
| At least one item per required kind | All | No. |
| Extra evidence allowed | Multiple quote/invoice links | No. |
| Free text cannot replace required file evidence | 4, 5 | No. |
| File URL on returned run data | 4, 5 | No. |
| Uploaded files retained with run | Relevant to 4, 5; not time-tested | Keep; ownership of uploads never attached to a run remains unspecified. |
| MCP over Streamable HTTP | Assumed, not exercised | Not removable based on process examples. |
| OAuth bearer authentication | Assumed | No. |
| Trusted human presence | Human-assisted-client check | No; specify integration boundary. |

## Profiles

| Profile | Used? | Removal judgment |
|---|---|---|
| `parallel` | Required by 10 | Keep, but the one-line placeholder is insufficient. |
| `each` | Avoided by 6 and external tracking in 8 | Leave outside core. Needed only if native per-item ownership/tracking is a requirement. |
| `files` | None | Do not implement based on these processes. No scripts or templates are needed. |
| `rules` | Would enforce thresholds in 1, but core explicitly delegates them | Leave outside core. Necessary for organizations demanding server-enforced routing. |
| `subprocess` | None | Do not add to satisfy these ten cases. |
| `evaluation` | None | Do not add. Human review and objective submission checks cover these examples. |
| `actions` | None | Do not add solely to disguise external side-effect uncertainty. Existing systems and instructions can manage it for these cases. |

The defensible removals are small redundant result flags, not whole workflow primitives. Most unused features are either operational safeguards or deliberately optional profiles. Ten examples do not justify stripping them indiscriminately.

# Task 4 — Adoption verdict

An organization could adopt this as a lightweight coordinator for mostly sequential, human-supervised processes, provided it accepts that routing correctness, authority, business validation and external action safety remain with workers and existing systems. It is not yet a credible replacement for the organization’s general workflow control layer: the explicit major-incident requirement has no executable parallel specification, human task results omit the schemas needed to complete them, and several edge contracts disagree or stop before recovery is defined. An agent that has never seen these documents can plausibly discover, claim and complete an ordinary agent step from tool results, because `claim` carries instructions and acceptance requirements; it cannot reliably navigate every supported run from those results alone. Fix the missing human schemas, replay precedence, approval-branch rules, event ordering and snapshot refresh semantics, then publish a real parallel contract. No larger feature catalogue is needed to address what these ten processes actually broke.