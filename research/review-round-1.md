The format can describe most of these processes. It does **not yet define an interoperable execution protocol**. Several essential behaviors depend on conventions that two servers could implement differently.

I wrote all ten complete `PROCESS.md` files, linked below. They use only section 2: no profiles, extensions, or invented fields. The walkthroughs are simulations, not executions against a server.

One immediate defect: **the specification’s own example declines an approved supplier.** `approve` has no `next`, so under “Default: the following step in the list,” approval advances to `decline`. `done` is unreachable, and no step creates the supplier record promised in the body.

## Task 1 — Ten processes

### How to read the walkthroughs

The draft does not provide complete request and response schemas. Pretending otherwise would hide one of its largest defects.

For the walkthroughs, I assume these illustrative wire conventions:

| Notation | Calls, in order, and returned data |
|---|---|
| `START(process, inputs)` | `start_run({process, inputs, requestId})` → `{runId, state, currentStep}`. These response fields are **assumed**, not specified. |
| `A(step)` | `get_work()` → ready item with exact `claim` arguments; `claim({runId, stepId, requestId})` → `{token, step, body, runData, files: []}`. `runData` contains the specified inputs and accumulated outputs and decisions below. |
| `S(output, evidence)` | `submit({token, output, evidence, requestId})` → assumed acceptance receipt `{accepted: true}`. The next entry shows the resulting work. |
| `H(step, decision, output?)` | Human calls `get_run({runId})` → run data, states, and a work item with snapshot `s1`, `s2`, etc.; then `decide({workItemId, snapshot, decision, output?, evidence?, requestId})` → assumed acceptance receipt. **The `decide` argument names are illustrative.** |
| `F(file)` | `upload({fileName, contentType, contentBase64, requestId})` → `{id, sha256}`. `requestId` is required by the universal write rule but missing from the tool’s listed arguments. |
| `END(outcome)` | `get_run({runId})` → `{state: "ended", outcome, ...recordedRunData}`. |

Evidence below uses illustrative `{kind, reference}` items. The specification defines their meaning, **not this serialization**.

Every write gets a distinct request ID; an exact retry reuses that ID. External application calls are illustrative tools belonging to the agent, not additional process-server tools.

**First shared uncertainty:** discovery already stops short. `describe` gives no exact capability schema, and `list_processes` returns no input schema. Assuming the caller already knows the process inputs, `start_run` then has no specified response or explicit initial-step rule. Every walkthrough below continues on the assumption that it returns a run ID and activates the first listed step. I mark the next process-specific gap separately.

### 1. Invoice approval above 10k and 100k

**Complete file:** [invoice-approval/PROCESS.md](../conformance/fixtures/invoice-approval/PROCESS.md)

**Verdict: fits cleanly as an instructed workflow.** It does not provide server-enforced monetary routing.

The company policy in this file is explicit:

- Total includes tax and is expressed in USD.
- Up to and including $10,000: manager.
- Above $10,000 through $100,000: controller.
- Above $100,000: CFO.
- These are replacement tiers, not cumulative approvals.

Those choices matter. “Different approvers above 10k and above 100k” does not itself settle currency, tax, boundaries, or cumulative approval.

The routing instructions are:

```yaml
      Choose manager_review for totals up to and including 10000;
      choose finance_review for totals above 10000 up to and including 100000;
      choose cfo_review for totals above 100000. Explain the amount and choice.
```

Each approval also states its permitted amount range. This is expressible with instructions; I would not add a rules engine just to write it.

**Agent run**

1. `START(invoice-approval, {invoiceId:"INV-284", invoiceDocument:"DMS:INV-284", manager:"person:mira"})` → run `I1`, `validate` ready.
2. `A(validate)` → token `i1`, full invoice instructions and those inputs.
3. External `erp.get_invoice("INV-284")` → supplier Acme, USD, total `125000`, PO `PO-88`.
4. External `erp.match_invoice("INV-284")` → receipt matched, no duplicate, report `ERP:MATCH-284`.
5. Attempt `S({totalUsd:125000, supplier:"Acme", matchSummary:"PO and receipt matched; no duplicate"}, links to invoice and match report)`, with `next:"cfo_review"` and `reason:"125000 exceeds 100000"`.

   **First process-specific gap:** §3.2 says the submission “MUST include `next` … and `reason`,” but §4.1 defines `submit` as `{token, output, evidence?, requestId}`. There is no specified place for either routing field. Continue assuming top-level fields are accepted.

6. `H(cfo_review, approved)` → CFO decision recorded; `record` ready.
7. `A(record)` → token `i2`, including the CFO decision.
8. External `erp.find_approval("INV-284")` → none.
9. External `erp.record_approval(...)` → `ERP:AP-901`.
10. `S({approvalRecord:"ERP:AP-901"}, link)`; `END(approved)`.

**Approver run**

The CFO opens `I1`, checks invoice, matched receipt, amount, and expenditure justification, then approves with the current snapshot.

A real CFO would reject a screen that does not identify the exact invoice and amount being approved. The draft guarantees run data, but not an approval presentation or document retrieval mechanism.

Also: the server can accept a wrong tier if the agent chooses an allowed ID. The approval wording catches that only if the human follows it. Do not sell this as an enforced delegation-of-authority matrix.

**Addition needed:** none to express this process. The wire contradiction needs correction.

### 2. Employee onboarding across HR, IT, and the manager

**Complete file:** [employee-onboarding/PROCESS.md](../conformance/fixtures/employee-onboarding/PROCESS.md)

**Verdict: fits cleanly.**

HR → manager access plan → IT provisioning → agent checklist → manager readiness approval is a legitimate sequential onboarding process. The request did not require simultaneous work.

**Agent run, including intervening human work**

1. `START(employee-onboarding, {employeeId:"E-412", employeeName:"Nina Rao", startDate:"2026-11-02", hiringManager:"person:mira"})` → `O1`, HR work pending.
2. `H(hr_setup, completed, {hrisRecord:"HR:E-412", location:"London", jobTitle:"Analyst"})`, evidence `HR:E-412`.
3. `H(access_plan, completed, {accessRequests:["CRM:read","Analytics:standard"], equipment:"Standard laptop", buddy:"person:jo"})`.
4. `H(it_setup, completed, {assetTag:"LT-440", accounts:["CRM:nrao","Analytics:nrao"], handoverPlan:"Collect laptop Monday at reception"})`, evidence `IT:771`.
5. `A(welcome)` → token `o1`, inputs and all three human outputs.
6. External `onboarding.find_checklist("E-412")` → none.
7. External `onboarding.create_checklist(...)` → `ONBOARD:412`.
8. `S({checklist:"ONBOARD:412"}, link)`.
9. `H(confirm_ready, approved)`; `END(ready)`.

**First process-specific gap:** the HR task is assigned to a role, and §7 allows mapping roles “to people or groups.” It does not say whether a group gets a shared claimable item, one selected recipient, or some other assignment behavior.

**Approver run**

Mira sees the HR reference, equipment, account list, buddy, and checklist. She rejects if access is wrong. `on_reject: access_plan` repeats access planning, IT setup, and checklist preparation.

The manager would refuse to act on stale IT output presented as current during that repeat. The spec does not define invalidation of downstream outputs after rejection.

HR would also refuse putting identity documents into this shared run. The file explicitly keeps them in HRIS because §3.1 says everyone doing a step “sees the whole run data.”

**Addition needed:** none for this scoped process. Assignment and rejection semantics need definition.

### 3. Customer support escalation to engineering

**Complete file:** [support-escalation/PROCESS.md](../conformance/fixtures/support-escalation/PROCESS.md)

**Verdict: fits cleanly.**

Engineering involvement is ordinary declared human work. It should not be confused with the `escalate` tool, which is an agent’s inability to finish its current step.

**Agent run**

1. `START(support-escalation, {ticketId:"SUP-991"})` → `S1`.
2. `A(diagnose)` → token `s1`.
3. External `support.get_ticket("SUP-991")` → exports fail for large reports.
4. External `diagnostics.read(...)` → timeout after 60 seconds.
5. External `knowledge.search(...)` → no approved fix.
6. `S({diagnosis:"Export worker times out on large reports", attemptedFixes:"Retried with approved reduced batch size; still fails"}, diagnostic link)`, choosing `tier1_handoff`, reason `"No approved tier-1 resolution"`.

   **First process-specific gap:** the same missing routing fields in `submit` identified in process 1.

7. `H(tier1_handoff, completed, {reproduction:"Export account 17 report for one year", impact:"Customer cannot complete monthly reporting", severity:"high"})`, ticket link.
8. `H(engineering, completed, {resolution:"Worker timeout corrected in release 8.4.2", issue:"ENG-731"})`, deployment link.
9. `H(confirm, approved)` after customer confirmation.
10. `A(close_ticket)` → token `s2`, engineering resolution and confirmation.
11. External `support.close("SUP-991", ...)` → ticket closed.
12. `S({ticket:"SUP-991"}, link)`; `END(resolved)`.

**Approver run**

The tier-1 lead reviews the reproduction before handoff, then later confirms the result with the customer. If the customer still sees failures, rejection returns to handoff and engineering.

The lead would refuse an engineering task labelled complete merely because an issue was filed. The file expressly keeps it open until a fix or agreed workaround exists; no format addition is needed.

If the agent instead calls `escalate`, the draft does not say **who receives that escalation**. It cannot be relied on as automatic engineering routing.

**Addition needed:** none.

### 4. Legal and finance contract review before signature

**Complete file:** [contract-review/PROCESS.md](../conformance/fixtures/contract-review/PROCESS.md)

**Verdict: fits cleanly.**

Sequential legal and finance review satisfies the request. Both review the same immutable revision. Either rejection returns to preparation and repeats both reviews.

**Run**

This process deliberately contains no agent work. An agent can initiate and observe it without pretending to be a reviewer.

1. `START(contract-review, {contractId:"C-77", document:"DMS:C-77", owner:"person:lee"})` → `C1`.
2. Owner calls `get_run` → preparation item, snapshot `c1`.
3. `F("C-77-r3.pdf")` → `{id:"file-c3", sha256:"<hash of r3>"}`.

   **First process-specific gap:** §4 requires every write to carry `requestId`, but `upload` omits it. The file evidence item’s wire shape is also unspecified.

4. Owner calls `decide` with `completed`, snapshot `c1`, output `{revision:"C-77/r3", summary:"USD 48,000 annual subscription; net 30; agreed liability cap"}`, evidence `file-c3` → accepted.
5. `H(legal_review, approved)` for revision r3.
6. `H(finance_review, approved)` for revision r3.
7. Signatory retrieves the signed agreement from the signature service; `F("C-77-r3-executed.pdf")` → `file-c3-signed`, its hash.
8. `H(sign, completed, {signedRevision:"C-77/r3", signedAt:"2026-10-09T10:00:00Z"})`, signed file evidence.
9. `END(signed)`.

The initiating agent can call `get_work`; it receives no work for this run.

**Approver run**

Legal reviews r3 and approves. Finance sees that legal approval and the same r3 attachment. If finance requests a changed payment term, it rejects; preparation produces r4 and legal reviews again.

A lawyer or finance reviewer would refuse a “latest contract” URL whose contents can change after approval. The file prevents this by instruction and file attachment.

A signatory would refuse a system that cannot show which document version each approval covered. Snapshots protect against changes to what the server tracks; they do not themselves bind a signature to an external document.

**Addition needed:** none to express the review. Attachment access and approval records require precise semantics.

### 5. Expense reimbursement with receipts and manager approval

**Complete file:** [expense-reimbursement/PROCESS.md](../conformance/fixtures/expense-reimbursement/PROCESS.md)

**Verdict: fits cleanly.**

An employee supplies receipts, the agent checks them, the manager approves, and accounts payable pays.

**Agent run**

1. `START(expense-reimbursement, {claimId:"EX-19", manager:"person:mira"})` → `E1`.
2. Employee opens the `receipts` item; `F("EX-19-receipts.pdf")` → `file-e19`, hash.

   **First process-specific gap:** the upload/idempotency and evidence serialization gaps described above.

3. `H(receipts, completed, {purpose:"Customer site visit", totalUsd:240, lines:[{merchant:"Rail", date:"2026-10-05", amount:160, purpose:"Travel"},{merchant:"Hotel", date:"2026-10-05", amount:80, purpose:"Overnight stay"}]})`, receipt file.
4. `A(check)` → token `e1`, receipt reference and submitted lines.
5. External `expenses.get_policy()` → applicable travel policy.
6. External `expenses.find_duplicates("EX-19", ...)` → none.
7. External receipt reader → two receipts, amounts 160 and 80, matching merchants and dates.

   **Further blocking gap:** the draft has no evidence-file retrieval tool. Its `download` returns a “process file,” not an uploaded receipt. Continue assuming the claim’s evidence references can be opened through an implementation-specific channel.

8. `S({checkedTotalUsd:240, checks:"Both receipts matched; sum correct; policy compliant; no duplicate"}, check-report link)`.
9. `H(manager_review, approved)`.
10. `H(reimburse, completed, {paymentReference:"PAY-881", paidUsd:240})`, settled-payment link.
11. `END(reimbursed)`.

**Approver run**

Mira opens the receipts and business purpose, checks the amount, and approves. Missing or personal expenses cause rejection back to `receipts`.

She would refuse a system that shows only “one file attached” without letting her inspect the receipts. Section 6 guarantees evidence kind and verification, not receipt coverage.

Accounts payable would refuse to treat an arbitrary submitted `manager` as authoritative. The file says not to approve one’s own claim, but manager identity and authorization need organization policy outside the format.

**Addition needed:** none to express it. Do not confuse its instructions with server validation of every receipt or the employee’s reporting line.

### 6. RFP with three supplier quotes

**Complete file:** [supplier-rfp/PROCESS.md](../conformance/fixtures/supplier-rfp/PROCESS.md)

**Verdict: fits cleanly.**

One buyer can collect three quotes in one human task. The request does not require separate supplier work items, independent timers, or a supplier portal.

**Agent run**

1. `START(supplier-rfp, {rfpId:"RFP-8", requirements:"Twenty laptops, delivered within four weeks", supplierOne:"Acme", supplierTwo:"Beacon", supplierThree:"Cedar"})` → `R1`.
2. Buyer requests quotes through their normal supplier channels.
3. Three `upload` calls → `file-acme`, `file-beacon`, `file-cedar`, each with a hash.
4. `H(collect_quotes, completed, {quoteOne:"ACME-Q10", quoteTwo:"BEACON-Q8", quoteThree:"CEDAR-Q41"})`, all three files.

   **First process-specific gap:** upload/evidence encoding. The server also has no declared cardinality relationship between these three output fields and the attachments.

5. `A(compare)` → token `r1`, quote references and evidence.
6. External document reader, assuming attachment access → Acme `$24,000/4 weeks`; Beacon `$25,000/3 weeks`; Cedar `$23,000/7 weeks`.
7. `S({comparison:"Acme and Beacon meet delivery; Cedar does not", recommendedSupplier:"Acme", rationale:"Lowest price among compliant quotes"}, [])`.
8. `H(select, approved)`.
9. `A(record_selection)` → token `r2`.
10. External `sourcing.find_selection("RFP-8")` → none.
11. External `sourcing.record_selection(...)` → `SOURCE:RFP-8`.
12. `S({sourcingRecord:"SOURCE:RFP-8"}, link)`; `END(selected)`.

**Approver run**

Procurement reviews all three quotes, delivery differences, and the recommendation. It approves Acme or rejects for rework.

The manager would refuse a comparison based on three quote IDs but only one actual quote. The agent instructions require all three; `evidence: [file]` does not.

Rejection returns to quote collection even if only the comparison needs correction. That is a coarse authoring choice, not a missing protocol feature: another ordinary step could separate correction paths if the company needs it.

**Addition needed:** none.

### 7. Blog draft through editor review to publication

**Complete file:** [blog-publication/PROCESS.md](../conformance/fixtures/blog-publication/PROCESS.md)

**Verdict: fits cleanly.**

**Agent run**

1. `START(blog-publication, {brief:"Explain the new reporting feature", slug:"reporting-update", publishAt:"2026-10-12T09:00:00Z"})` → `B1`.
2. `A(draft)` → token `b1`.
3. External `cms.save_draft(...)` → immutable revision `POST-14/r1`, preview URL.
4. `S({revision:"POST-14/r1", title:"Reporting update", summary:"New reports and scheduling"}, preview link)`.
5. `H(editor_review, rejected, note:"Correct the availability date")`.

   **First process-specific gap:** §3.2 says the rejection note is kept on the approval step, but specifies no field where the next agent can find it. Continue assuming `steps.editor_review.note`.

6. `A(draft)` → token `b2`, previous draft plus that note.
7. External `cms.save_draft(...)` → `POST-14/r2`.
8. `S({revision:"POST-14/r2", title:"Reporting update", summary:"Availability date corrected"}, r2 preview link)`.
9. `H(editor_review, approved)`.
10. Server waits until `2026-10-12T09:00:00Z`.
11. `A(publish)` → token `b3`, current r2 and approval.
12. External `cms.get_publication("reporting-update")` → unpublished.
13. External `cms.publish("POST-14/r2")` → public URL, revision r2.
14. External HTTP check → `200`.
15. `S({url:"https://company.example/blog/reporting-update", publishedRevision:"POST-14/r2"}, link)`; `END(published)`.

**Approver run**

The editor rejects r1, opens r2, checks the correction, and approves r2. The editor would refuse an interface that lets a stale approval publish a later unreviewed CMS revision.

The file instructs the publishing agent to check the approved revision. That is sufficient to express the requirement; it does not make the server verify CMS state.

If approval arrives after the scheduled timestamp, the file says to publish immediately. The server still needs a defined past-time rule for `wait_until`.

**Addition needed:** none.

### 8. Incident postmortem with actions tracked to closure

**Complete file:** [incident-postmortem/PROCESS.md](../conformance/fixtures/incident-postmortem/PROCESS.md)

**Verdict: fits with a workaround.**

The core can keep the run open until a person confirms all actions closed. It cannot represent an arbitrary number of individually assigned, independently progressing action tasks using section 2.

**Exact workaround lines**

```yaml
      Create or find one issue per approved action in the existing issue tracker,
      linked to incidentId. Set its owner and due date.
```

```yaml
      Track every action in the issue tracker. Chase owners and overdue work there.
      Keep this task open until every action has closure evidence meeting its
      approved completion criterion.
```

And the body states:

> “The issue tracker owns per-action assignments, due dates and progress; this run has one aggregate closure task.”

**What a process owner would find wrong**

The process server cannot answer “Which action is overdue, who owns it, and what remains?” from its native work items. It sees one incident-owner task. Action-level accountability lives in another system.

That is a usable workaround for a company already using an issue tracker. It is not native action tracking.

**Agent run**

1. `START(incident-postmortem, {incidentId:"INC-44", incidentOwner:"person:jo"})` → `P1`.
2. `A(draft)` → token `p1`.
3. External `incidents.read("INC-44")` → timeline, impact, recovery record.
4. `S({report:"DOC:PM-44", proposedActions:[{deliverable:"Add queue-depth alert", owner:"person:alex", due:"2026-10-20", criterion:"Alert fires in controlled test"},{deliverable:"Add rollback drill", owner:"person:sam", due:"2026-10-27", criterion:"Successful timed drill recorded"}]}, report link)`.
5. `H(review, approved)`.
6. `A(register_actions)` → token `p2`.
7. External `issues.find_by_incident("INC-44")` → none.
8. External `issues.create(...)` → `ACT-1`.
9. External `issues.create(...)` → `ACT-2`.
10. `S({actionIds:["ACT-1","ACT-2"], trackerView:"TRACKER:INC-44"}, link)`.
11. Incident owner later calls `H(track_closure, completed, {closures:[{issueId:"ACT-1", evidence:"TEST:ALERT-7"},{issueId:"ACT-2", evidence:"DRILL:29"}], closureSummary:"Both completion criteria verified"})`, tracker link.
12. `END(closed)`.

**First process-specific gap:** at the human review, the snapshot must represent “anything the person was looking at.” The report is a link. The spec does not say whether external document changes invalidate the snapshot. Continue assuming an immutable report revision.

**Approver run**

Jo approves the report and proposed actions, then remains responsible for the aggregate closure task. Individual owners work in the tracker.

Jo would refuse a dashboard claiming “all actions complete” merely because `closures` is a list. Section 6 does not enforce one closure per action or validate completion criteria. The task instructions make Jo responsible for that check.

**Addition needed:** none for the requested outcome with this workaround. If action assignments and status must live inside the protocol, the smallest relevant addition is the already-listed `each` capability, including per-item identity and state. That stronger requirement was not stated, so I am not making it a core requirement.

### 9. Customer offboarding

**Complete file:** [customer-offboarding/PROCESS.md](../conformance/fixtures/customer-offboarding/PROCESS.md)

**Verdict: fits cleanly as a sequential business process. Event execution remains underspecified.**

The file covers cancellation, final billing, settlement, authorized deletion, and verification. It does not call deletion complete while scheduled expiry is still pending.

**Agent run**

1. `START(customer-offboarding, {customerId:"CUS-32", effectiveAt:"2026-10-31T18:00:00Z"})` → `F1`.
2. `H(authorize, approved)`.
3. Server reaches `effective_time`.

   **First process-specific gap:** the `datetime` representation and treatment of already-past timestamps are unspecified. Continue assuming RFC 3339 and immediate continuation when overdue.

4. At the effective time, `A(cancel_services)` → token `f1`.
5. External `customers.get_export_status("CUS-32")` → handed over.
6. External `services.list("CUS-32")` → two active services.
7. Two external cancellation calls → `CANCEL-1`, `CANCEL-2`.
8. `S({cancellationIds:["CANCEL-1","CANCEL-2"]}, cancellation links)`.
9. `A(final_invoice)` → token `f2`.
10. External `billing.finalize("CUS-32")` → invoice `FINAL-32`, balance `420`, USD.
11. `S({invoiceId:"FINAL-32", balance:420, currency:"USD"}, invoice link)`, choosing `settlement_event`, reason `"Outstanding final balance"`. This encounters the routing-field contradiction.
12. Billing system later calls `send_event({runId:"F1", name:"final-invoice-settled", data:{invoiceId:"FINAL-32"}, requestId})` → assumed delivery acknowledgement.
13. `A(verify_settlement)` → token `f3`.
14. External `billing.get_invoice("FINAL-32")` → settled, balance zero.
15. `S({ledgerSummary:"FINAL-32 settled in full"}, ledger link)`, choosing `deletion_clearance`.
16. `H(deletion_clearance, completed, {deletionScope:"Customer workspace and application exports", retentionExceptions:"Billing records retained under approved schedule"})`, inventory link.
17. `A(delete_data)` → token `f4`.
18. External deletion tools → `DELETE:APP-32`, `EXPIRE:BACKUP-32`.
19. `S({deletionReceipts:["DELETE:APP-32","EXPIRE:BACKUP-32"]}, receipt links)`.
20. After expiry is verified, `H(verify_deletion, completed, {completionStatement:"Approved deletion completed; retained billing records listed"})`, verification link.
21. `END(offboarded)`.

**Approver run**

The account owner authorizes the customer request and export arrangement. The data steward later specifies scope and retention exceptions, then verifies completion.

The steward would refuse an “offboarded” outcome supported only by a deletion-job submission receipt. The separate verification task fixes that with existing syntax.

A billing event arriving before `settlement_event` becomes active may be lost, buffered, or rejected: unspecified. The timeout to collections and ledger check provide a recovery path, but a paid customer could still wait thirty days unnecessarily.

**Addition needed:** none to express the process. Specify event delivery semantics. A manual finance task can replace the event if necessary; no new feature is required.

### 10. Major incident response with concurrent accountable teams

**Complete section-2 attempt:** [major-incident-response/PROCESS.md](../conformance/fixtures/major-incident-response/PROCESS.md)

**Verdict: does not fit.**

The actual requirement is that operations, security, and communications receive independent work items at activation, work concurrently, have deadlines measured from activation, and report separate live progress. The commander closes only after all three finish.

**Exact failed workaround**

```yaml
      Open or find the incident bridge and page operations, security and customer
      communications together. Tell all three to start immediately and record
      their progress in the incident system.
```

This is followed by ordinary sequential `operations`, `security`, and `communications` tasks.

Paging three people does not create three independent process work items. The body states the unmet requirement explicitly rather than pretending the attempt implements it.

**Agent run**

1. `START(major-incident-response, {incidentId:"SEV-7", commander:"person:jo"})` → `M1`.
2. `A(activate)` → token `m1`.
3. External `incidents.open_bridge("SEV-7")` → `BRIDGE:7`.
4. External `paging.page_teams(...)` → receipt `PAGE:7`, all three teams notified.
5. `S({bridge:"BRIDGE:7", pagingReceipt:"PAGE:7"}, links)`.
6. Server creates only the operations work item.
7. Security attempts to record containment completion while operations continues: **no security work item exists.**
8. After operations completes, security becomes ready; after security completes, communications becomes ready; then commander approval can occur.

**First process-specific point the spec does not supply:** assignment behavior for the role-based operations item is undefined, as in onboarding. The more important failure at step 7 is **not ambiguity**: the required concurrent work does not exist under the declared sequential structure.

**Approver run**

The commander opens the incident while operations is working. Security and communications have no independently active items, snapshots, or completion channel. Their `due: 1h` clocks start only when their later steps become ready, not at incident activation.

A real incident commander would refuse this as the system of record for live coordination. Retrospectively clicking through finished work falsifies the operational timeline.

**Smallest addition**

A bounded parallel block: activate these three existing task branches together, track their separate states, and join only after all complete. Define what happens to siblings on failure or cancellation.

That is the already-advertised `parallel` profile. It needs actual semantics. No expression language, arbitrary graph engine, or server-executed actions are necessary.

## Task 2 — Behavioral ambiguities and omissions

These are differences in observable behavior, not writing preferences. Where clauses conflict, the alternatives below represent different ways an implementer could resolve that conflict; they are not both compliant with every literal clause.

### Publication, parsing, and schema

| # | Spec basis | Behavior A | Behavior B |
|---|---|---|---|
| 1 | §2 shows YAML-like frontmatter but specifies no parser/version. | YAML scalar `yes` is a boolean. | It is a string. |
| 2 | No duplicate-key rule. | Duplicate `next` or input keys are rejected. | The last value wins. |
| 3 | `name` is “Same as the folder name.” | Name mismatch rejects import. | Import renames the folder to the declared name. |
| 4 | “Unknown fields are refused.” Scope is unstated. | Unknown keys are rejected in field definitions and every structural object. | Only unknown top-level and step keys are rejected. |
| 5 | `x-` fields MUST be kept and MAY be ignored. | They are retained and exposed in claims/export. | They are retained internally but omitted from claims/export. |
| 6 | `version` is the author’s string; publication creates a “numbered version.” | Catalog `version` is the author’s string. | Catalog `version` is the server revision number. |
| 7 | `start_run` takes `process`, with no version selector definition. | It resolves the latest published version. | It accepts a revision-qualified identifier and can start old versions. |
| 8 | `list_processes` promises name, description, version only. | It additionally exposes input schemas. | The caller must obtain schemas elsewhere. A new agent cannot prepare valid inputs from the catalog alone. |
| 9 | `one_of` is listed, but acceptance guarantees mention declared type, not enum membership. | Out-of-enum values are rejected. | Type-correct values are accepted and enum guidance is left to the actor. |
| 10 | `items` is listed without a grammar. | `items: object` and recursive schema objects are supported. | Only a scalar type name is supported. |
| 11 | `optional: true` is defined, nullability is not. | Optional means omission only; `null` is invalid. | Optional fields may also be null. |
| 12 | `date` and `datetime` have no encoding or timezone rules. | Only strict ISO dates and timezone-bearing timestamps pass. | Local or implementation-native timestamp strings pass. |
| 13 | `number` has no range or precision semantics. | Values are processed as binary floating point. | Values retain decimal precision. Monetary boundary comparisons can differ. |
| 14 | `person` has no serialized representation. | A person is an organization identity ID. | Email strings or identity objects are accepted. |
| 15 | `object` has no further structure. | Any JSON object passes. | Only platform-specific object representations pass. |
| 16 | “A submission MAY contain extra fields; a server keeps them.” | Only extra output fields are allowed. | Extra submission-envelope fields are also stored. |
| 17 | Extra fields are mentioned for submissions, not starting inputs. | Undeclared inputs are rejected. | Undeclared inputs are retained as business metadata. |
| 18 | Step keys have applicability rules but little cross-field validation. | `on_timeout` without `timeout`, or `overdue` without `due`, rejects publication. | Such fields are accepted but never used. |
| 19 | Cycle prohibition is phrased as a path revisiting a step “only via” rejection or overdue. | A rejection followed by replay of the normal forward path is allowed. | Literal repeated visits on the forward path are considered forbidden. The examples imply A, but the rule needs a graph definition. |
| 20 | No reachability validation rule. | Unreachable steps are allowed. | Publication rejects them. The supplied `done` step exposes this difference. |

### Routing, lifecycle, and retained data

| # | Spec basis | Behavior A | Behavior B |
|---|---|---|---|
| 21 | No explicit entry-point rule; §6 requires a completed predecessor. | The first step is implicitly ready at start. | An implementation introduces a synthetic start transition or rejects a process lacking its expected entry convention. |
| 22 | A last non-finish step defaults to “the following step.” | Completing it ends the run with a default outcome. | Publication or completion fails because no successor exists. |
| 23 | §3.2 requires chosen `next` and `reason`; `submit` omits both. | They are top-level arguments. | They are embedded in `output`, or the advertised schema rejects them. |
| 24 | `next` lists apply to “all but finish,” including server steps. | A timer with multiple successors is rejected at publication. | The server chooses a successor using an implementation convention. There is no submitting actor. |
| 25 | Human tasks and approvals may also have list-valued `next`. | `decide` accepts a choice and reason. | The human channel has no routing fields and cannot complete them. |
| 26 | No rule for a supplied choice on scalar `next`. | An unnecessary matching `next` is accepted. | Any supplied routing choice is invalid. |
| 27 | “`reason: <text>`” has no content rule. | Empty text passes. | Empty or whitespace-only reasons fail. |
| 28 | §3.1 exposes `output`, `evidence`, and `by` for completed steps. | Approvals have explicit decision, note, and timestamp fields. | Those facts appear only in an implementation-specific audit record. |
| 29 | Rejection note is “kept on the approval step.” | It is `steps.id.note`. | It appears inside output or history. Agents cannot use one portable path. |
| 30 | Repeat history is specified “after `on_reject`.” | Overdue and retry repetitions also append history. | They replace current state without the same history structure. |
| 31 | No history-entry structure or ordering. | Entries are chronological full completions. | Entries are reverse chronological partial deltas. |
| 32 | No downstream invalidation rule after rejection. | Later outputs remain visible as current until replaced. | Outputs from the abandoned pass are marked invalid or removed from current data. |
| 33 | “Latest completion is current” while rework is unfinished. | An earlier completion stays current during a new attempt. | Reopening clears current output until recompletion. |
| 34 | Run is `waiting` when “only people, timers or events are pending.” | An escalated agent step makes the run waiting. | It remains active because its declared kind is still agent. |
| 35 | Step state `waiting` is listed without kind-specific transitions. | Human work is immediately waiting. | It is ready until a person opens or accepts it. |
| 36 | Operator may retry “a step that ended the run as `failed`.” | Any failed kind, including human tasks, is retried into its appropriate state. | Retry sets every kind to `ready` literally, producing different human-task behavior. |
| 37 | Retry does not define downstream or decision cleanup. | Failed-step decision remains current alongside the reopened item. | It moves to history and disappears from current state. |
| 38 | No cancellation transition table. | Cancel on an ended run is rejected. | It changes the run to cancelled. |
| 39 | Cancellation reason and partial data storage are unspecified. | Reason and completed outputs remain in `get_run`. | They are only available in a separate audit channel. |

### Claims, failures, and idempotency

| # | Spec basis | Behavior A | Behavior B |
|---|---|---|---|
| 40 | Claim duration is described, but claim expiry data is not. | Claim returns an expiry timestamp. | Caller must infer expiry from a server default and request timing. |
| 41 | Expired tokens are stale; requeue timing is not stated. | Step becomes immediately claimable on expiry. | A lease sweeper requeues it later. |
| 42 | `renew` returns a “new token.” | Renewal immediately invalidates the old token. | Both tokens remain usable until a defined overlap ends. |
| 43 | “`renew` extends it” does not define the basis. | New expiry is now plus lease duration. | New expiry is previous expiry plus lease duration, potentially allowing accumulation. |
| 44 | `renew` is optional; no long-work convention. | Long tasks must repeatedly checkpoint outside the protocol. | Server leases are sufficiently long and extension is unnecessary. The agent cannot infer a portable strategy. |
| 45 | §6 refers to work being “released”; no release operation exists. | Server exposes an extra release tool. | Clients must wait for lease expiry. |
| 46 | Escalation has no assignee field or role rule. | It goes to the run initiator. | It goes to an operations queue. |
| 47 | Escalation changes ownership, but detailed state/token behavior is absent. | Token becomes stale immediately and the step becomes waiting. | Token is invalidated but an internal claimed state persists until handoff processing. |
| 48 | There is no documented agent `fail` decision. | `escalate` is mandatory for every unrecoverable agent failure. | An implementation adds a failure operation or timeout-to-failure convention. |
| 49 | No `requestId` scope or lifetime. | IDs are unique per caller and expire after a day. | IDs are unique organization-wide and retained indefinitely. |
| 50 | Same ID with different arguments is unspecified. | Return conflict. | Return the original result without examining the new arguments. |
| 51 | Every write is idempotent; §4.3 says retry transient errors with the same ID. | Transient errors are not cached and a retry can succeed. | Every response is cached and the same transient error is replayed indefinitely. |
| 52 | Lost claim response returns the “same result, token included.” | After lease expiry, replay still returns the original successful response and expired token. | Server returns stale or includes additional expiry status. The interaction of the two rules is unresolved. |
| 53 | `upload` omits `requestId`. | Server requires an unlisted argument. | Upload is treated as an exception and can duplicate stored files. |
| 54 | Simultaneous writes are server-resolved, without precedence. | A submission committed at a deadline wins over overdue withdrawal. | The deadline transition wins and the submission is stale. |
| 55 | A post-submit external-effect receipt is not atomic with the effect. | A replacement agent is shown only process output and may repeat the effect. | Server includes additional attempt/progress information. Neither gives external exactly-once execution by itself. |
| 56 | Test mode prohibits real agent effects but does not specify other paths. | Human channels, notifications, and event integrations are also isolated. | Only the agent’s behavior changes; humans can still perform real actions. |

### People, identity, and visibility

| # | Spec basis | Behavior A | Behavior B |
|---|---|---|---|
| 57 | Roles map to “people or groups”; one work item is created. | A shared group item can be completed by any member. | One member is selected as the assignee. |
| 58 | No rule for role membership changes during a run. | Assignment is frozen at activation. | Current group membership determines who can decide. |
| 59 | A `person` string can be a role or a path. | Dot-containing values are interpreted as paths. | A matching local role takes precedence. |
| 60 | Path grammar is absent. | Dotted traversal only. | Escaped keys, array indices, or other path forms are supported. |
| 61 | Missing or invalid person path is unspecified. | Activation fails the run. | Work is placed in an unassigned administrative queue. |
| 62 | `initiator` means whoever started the run, including agents. | An agent-started initiator task is rejected or unassignable. | It is routed to the agent’s sponsoring human. |
| 63 | “A human-presence assertion the server trusts.” | A recent interactive login is enough. | Every decision requires fresh explicit confirmation. |
| 64 | `get_run` promises data, states, and work items, not task instructions/body. | Work items include instructions and approval text. | A separate UI must retrieve them; tool-only people cannot see what to do. |
| 65 | Snapshot covers “anything the person was looking at.” | Any run-data change invalidates it. | Only the target item and referenced fields invalidate it. |
| 66 | External links are evidence, but snapshot coverage is unstated. | Only the stored URL is covered. | The server also versions or hashes fetched document content. |
| 67 | Person completion of an escalated agent step lists `output`, not evidence. | Original evidence requirements still apply. | Human completion bypasses or lacks a way to satisfy them. |
| 68 | No policy for who may resolve a supplied person reference. | Any valid organization person supplied at start is accepted. | The server enforces reporting-line or segregation-of-duties rules. |
| 69 | Tools labelled “anyone” also operate “in this organization.” | Any authenticated organization member can read any run. | Object-level permissions limit visibility. |
| 70 | `get_work` returns work “the caller may claim,” but capability matching is absent. | All authorized agents see all available steps. | Work is filtered by tools, roles, or server-specific capability registrations. |

### Timers and events

| # | Spec basis | Behavior A | Behavior B |
|---|---|---|---|
| 71 | Duration syntax gives examples only. | Only positive integer `m`, `h`, `d` are accepted. | Seconds, zero, fractions, weeks, or composite durations are accepted. |
| 72 | `due` is measured after “ready”; human states are underspecified. | Clock starts at work-item creation. | Clock starts at assignment or acceptance. |
| 73 | Rework and renewal do not specify due-clock behavior. | Reopening restarts the due clock; renewal does not. | Due remains anchored to the first activation. |
| 74 | Overdue must “notify the organization.” | One notification to the assignee suffices. | Repeated notifications go to an organization escalation channel. |
| 75 | `overdue` says “pending work is withdrawn.” | A claimed agent’s token is immediately stale. | The claimed work is allowed to finish while the overdue path proceeds. The latter also conflicts with a strictly single-path interpretation. |
| 76 | `wait_until` references a value with no missing/past rule. | A past value continues immediately; missing data errors. | A past or missing value blocks for operator intervention. |
| 77 | `wait_for` continues when an event is delivered. | Early events are durably buffered for a later wait. | Events without an active matching wait are rejected or discarded. |
| 78 | Event `data` has no run-data location. | Payload is exposed as the wait step’s output. | It is retained only in an event log or not exposed at all. |
| 79 | Event names have no validation or matching rules. | Matching is case-sensitive. | Names are normalized before matching. |
| 80 | Repeated waits for the same event name have no occurrence identity. | A previously delivered event can satisfy a later occurrence. | Each delivery is consumed once. |
| 81 | A different request ID can carry the same business event. | Both deliveries are retained/consumed separately. | Business payload or event identity is deduplicated. |
| 82 | `on_timeout` defaults to “the following step”; `next` may point elsewhere. | Timeout uses list order regardless of `next`. | Implementer treats timeout as normal continuation through `next`. The text favors A, but the likely divergence must be made explicit. |
| 83 | No event-versus-timeout arbitration contract. | Event receipt before a clock boundary wins. | Transaction commit order decides. |
| 84 | No event payload validation or sender-to-event authorization model. | Any identity authorized for the run can send any event name. | Only configured integrations may emit particular events. |

### Evidence, files, packages, and conformance

| # | Spec basis | Behavior A | Behavior B |
|---|---|---|---|
| 85 | Evidence table defines kinds/references, not item JSON. | Items use `{kind, reference}`. | Items use `{type, id}` or keyed objects. |
| 86 | Required evidence is a list of kinds. | One item of each required kind is sufficient; additional items are allowed. | A server applies stricter multiplicity or rejects unexpected kinds. |
| 87 | File verification says “hash matches” without identifying the expected hash. | Server verifies current bytes against the upload-time stored hash. | Submitter must also supply a hash for comparison. |
| 88 | Uploaded evidence has no retrieval contract. | Claims/get_run include signed evidence URLs. | Only a proprietary UI can retrieve the upload ID. |
| 89 | `download` is optional, but all agents “can read every file.” | Claims include usable inline data or links when download is absent. | Claims expose paths only and some process files are inaccessible through the core tools. |
| 90 | Process-file paths have no normalization rules. | Only normalized relative paths are accepted. | Aliases, case-insensitive paths, or platform-specific forms work. |
| 91 | Upload ownership is organization-scoped; attachment authorization is unstated. | Any organization upload ID can be attached. | IDs are also restricted by uploader or run permissions. |
| 92 | Artifact retention and link expiry are unspecified. | Evidence remains retrievable for the life of the run record. | Uploaded evidence can expire while the record remains. |
| 93 | Hash covers only canonical frontmatter; “published version never changes.” | Body and `files/` are frozen separately despite not being hashed. | Only the hashed definition is treated as immutable; behavior-changing body/assets can change. The latter conflicts with the broad wording. |
| 94 | `x-content-hash` lies inside the hashed frontmatter. | Import removes this field before hashing. | It hashes the field too, producing a self-reference problem. |
| 95 | No import/export canonical representation. | YAML is parsed, validated, normalized, then canonicalized. | Parsed values are canonicalized before defaults or normalization. Hashes differ. |
| 96 | Optional tools may be required by a process, but publication checks mention only profiles. | A process using file evidence/events is rejected if upload/send_event is unavailable. | It publishes and later stalls. |
| 97 | Profile use has no declared syntax/version contract in core. | Server infers requirements from profile keys. | It expects an out-of-band manifest or extension field. |
| 98 | `list_runs` filters by “a business key given in inputs,” with no designation rule. | Any input field can be filtered. | Only one specially configured input is indexed as the business key. |
| 99 | “Oldest first” in `get_work` has no age basis or pagination. | Sort by current ready time; paginate. | Sort by run start time; return all available work. |
| 100 | “These five rules … [make a server] conformant,” plus housekeeping, versus other MUSTs and required tools. | Conformance requires the entire normative document. | Only §6 invariants and housekeeping are treated as the conformance test. |

A few issues above can be fixed by documenting an existing convention. Others need request/response schemas. None of that requires adding business-process features.

There are also **unambiguous limitations**, which should not be disguised as ambiguities:

- `link` evidence is deliberately unverified: §5 says “Nothing. Recorded as stated.”
- The core has no parallel work or native dynamic per-item tasks.
- `cancel` is not described as undoing external effects.
- Required output types do not prove business correctness.
- The supplied sample’s approved path reaches `decline`.

## Task 3 — What I would remove

“Used” below means used in the ten authored files, their bodies, or the demonstrated runs. Referenced recovery behavior is distinguished from an executed sample path.

An unused feature is a candidate, not automatic proof that it is unnecessary.

### Format fields and schema

| Field or construct | Used? | Remove? |
|---|---|---|
| `name` | All ten | No; stable process identity. |
| `description` | All ten | No; useful discovery information. |
| `inputs` | All ten | No. |
| `steps` | All ten | No. |
| Body | All ten | No; carries scope and operational instructions. Freeze it with the published version. |
| `version` author string | No | **Remove from minimal core**, or rename it to distinguish it from server revision. Two unrelated “versions” cause avoidable confusion. |
| `license` | No | **Remove from minimal core.** Package licensing can use a normal license file unless protocol consumers actually need this metadata. |
| `x-*` escape hatch | No | **Remove from minimal core.** None of these processes requires vendor fields; keep extensibility in explicitly versioned profiles if later needed. |
| `x-content-hash` | No | **Remove as currently specified.** It conflicts with its own hash scope. Retain server-generated publication hashes. |
| Step `id` | All ten | No. |
| `agent` | Nine; contract review needs no agent step | No. |
| `task` | Seven | No; human work is not always an approval. |
| `approve` | All ten | No. |
| `person` | All ten | No. |
| Role-name person assignment | All ten | No; useful portable assignment vocabulary. |
| Person-reference path | Invoice, onboarding, contracts, expenses, postmortem, incident response | No. |
| `initiator` | Expenses | No, but define agent-initiated behavior. |
| `output` | All ten | No. |
| `evidence` | All ten | No. |
| Scalar `next` | Invoice, offboarding | No; avoids unintended fall-through. |
| List `next` | Invoice, support, offboarding | No; fix its submission contract. |
| Default next-by-list-order | All ten | No; convenient, but the sample demonstrates its risk. |
| `on_reject` | Onboarding, support, contracts, expenses, RFP, blog, postmortem | No. |
| Default rejection outcome | Invoice, offboarding, incident response | No. |
| `due` | All ten | No; clarify clock and notifications. |
| `overdue` redirect | No | **Remove from minimal core.** None requires withdrawing pending work and changing paths merely because a deadline passed. Retain overdue status and notification. |
| `wait` duration step | No | **Remove from the smallest core for now.** No demonstrated need; scheduled publication and cancellation use timestamps. |
| `wait_until` | Blog, offboarding | No. |
| `wait_for` | Offboarding | No if events remain part of core; otherwise the manual finance task is a sufficient fallback. Do not leave event semantics half-defined. |
| `timeout` | Offboarding | No while `wait_for` remains. |
| `on_timeout` | Offboarding | No while `wait_for` remains. |
| `finish` | All ten | No; explicit outcomes are useful. |
| `string` | All ten | No. |
| `number` | Invoice, expenses, offboarding | No; specify numeric semantics. |
| `boolean` | No | Keep. A basic type costs little and avoids encoding yes/no as strings. No new feature proposal is involved. |
| `date` | Onboarding | No. |
| `datetime` | Contracts, blog, offboarding | No. |
| `list` | Onboarding, expenses, postmortem, offboarding | No. |
| `object` | Expense lines, postmortem actions and closures | No; be honest about the absence of nested-field validation. |
| `person` type | Several inputs and onboarding buddy | No. |
| Field-object `type` | Several | No. |
| Field `description` | No | Keep. Input discovery needs field explanations; these small examples used self-explanatory names. |
| `optional: true` | No | Keep. Forcing empty placeholder values would make real data worse. |
| `one_of` | Support severity | No; define enforcement. |
| `items` | Several list outputs | No; define its grammar. |
| `files/` assets | No | **Defer from the smallest core** unless there is an actual package asset use case. These ten use company systems and inline instructions. |
| `examples/<name>.yaml` | No | **Remove from core requirements.** They can remain an authoring convention. An expected outcome alone does not simulate human decisions or external systems. |
| Extension evidence kinds | No | **Defer.** Neither a vendor evidence framework nor its validation machinery is needed for these ten. |

### Tools

| Tool | Used? | Remove? |
|---|---|---|
| `describe` | Discovery convention for all runs | No; it needs an exact capability schema. |
| `list_processes` | Discovery convention | No; expose enough information to start a run. |
| `start_run` | All ten | No. |
| `get_run` | All ten | No. |
| `get_work` | Nine have agent work; contract run can be observed without it | No. |
| `claim` | Nine | No. |
| `submit` | Nine | No. |
| `escalate` | Explicit instructions in invoice, expenses, RFP, blog, offboarding; not exercised in sample paths | No; it is the only defined agent handoff when work cannot be finished. Define recipient and evidence behavior. |
| `cancel` | Blog rescheduling and offboarding body discuss it; no sample call | No; organizations need to stop requests. State that it does not compensate external effects. |
| `renew` | No sample needs it | Keep optional. A lease protocol without a way to finish legitimate long work is brittle. |
| `upload` | Contracts, expenses, RFP | No. Fix the request ID omission. |
| `download` | No process assets used | Defer with `files/`. It does **not** solve the demonstrated uploaded-evidence retrieval gap. |
| `decide` | All ten | **Keep as a required interoperable operation** if people must work through tools. Allow a UI to call the same semantics. |
| `send_event` | Offboarding | Keep with defined delivery semantics, or remove alongside event waits; do not claim events work merely because a tool exists. |
| `retry` | No sample path | **Defer from minimal core.** Its undefined reopening semantics are substantial. Ordinary rework already uses rejection. |
| `list_runs` | No sample path | Keep optional. Operational users need to find runs; ten single-run examples do not exercise administration. |

### Rules, guarantees, and packaging behavior

| Rule | Used or relied upon? | Remove? |
|---|---|---|
| Required `name`/`description`/`steps` | All ten | No. |
| Name and step-ID syntax | All ten | No; simple identifiers are useful. |
| Folder-name equality | All ten comply | Keep as a package rule; define mismatch handling. |
| Exactly one kind per step | All ten | No. |
| References must target existing steps | All ten | No. |
| Unknown-field rejection | All ten depend on correct interpretation | No; prevents silent typos. |
| Fields required by default | All ten | No. |
| Extra submitted fields retained | No | **Remove the obligation to retain arbitrary extras.** It permits undeclared data accumulation. If extras remain allowed, specify their scope and limits. |
| Ordinary cycles prohibited | All ten comply; no need to violate it | Keep for this core. Specify permitted rework using edge types, not ambiguous prose about revisits. |
| Whole-run visibility | All ten use or accommodate it | Keep for these files, but do not claim sensitive multi-department workflows automatically fit. |
| Latest output plus history | Blog rejection trace; several rejection paths | No. Define invalidation and attempt identity. |
| Chosen-route reason recorded | Invoice, support, offboarding | No. |
| Rejection note retained | Several; blog trace exercises it | No. |
| Claim token and lease | Nine | No. |
| One actor at a time | Nine agent flows; human decision races also need handling | No. |
| Idempotent writes | All ten | No. Clarify scope, retention, transient failures, and uploads. |
| Human-only task/approval/escalation completion | All ten | No. |
| Snapshot check | All ten | No. Define exactly what the snapshot covers. |
| Required-output acceptance | All ten | No. |
| Required-evidence acceptance | All ten | No. |
| `file` cannot be replaced with a text link | Contracts, expenses, RFP | No. |
| All validation issues returned together | No failed submission demonstrated | Keep; useful correction feedback with little conceptual cost. |
| `not_accepted` leaves state unchanged | Relied upon, not exercised | No. |
| `invalid` | No sample failure | Keep. |
| `stale` | Relied upon for snapshots and leases | Keep. |
| `conflict` | No sample race | Keep, but distinguish it from stale. |
| `forbidden` | Relied upon for human/agent separation | Keep. |
| `not_found` | No sample failure | Keep. |
| `retry` error | No sample failure | Keep; resolve its idempotency interaction. |
| Test-mode no-real-effects rule | No live sample calls were made | Keep, but define human/integration behavior. |
| Published version immutability | All ten rely on stable instructions | No. Extend integrity coverage to body/assets or clearly separate hash scope from version identity. |
| Run pins version | All ten | No. |
| Canonical JSON plus SHA-256 | No cross-server hash comparison demonstrated | Keep if content-addressed integrity is a requirement; fix the self-hash and normalization issues. |
| Server serializes concurrent writes | All ten rely on this under races | No. |
| Active/waiting/ended/cancelled run states | Walkthroughs rely on active, waiting, ended | Keep; specify transitions. |
| Ready/claimed/waiting/done/withdrawn/cancelled step states | First four used; latter two not exercised | Keep cancelled; **drop withdrawn if overdue redirects are removed**. |
| Import creates draft | Setup assumption only | Keep; avoids accidental activation of imported processes. |
| Import maps roles | All ten | No. |
| Publish only on request | Setup assumption | No; importing a file should not publish it automatically. |
| Package grants no permissions | Relied upon for role mapping and external tools | No. |
| Server does not execute package files | No package files used | Retain if file assets stay. |
| MCP/HTTP/authenticated identities | Assumed for all calls | No. Authentication alone does not specify run-level authorization. |

### Profiles

None was used in the ten files.

| Profile | Remove? |
|---|---|
| `parallel` | Keep outside core and specify it. Process 10 demonstrates an actual need. |
| `each` | Keep outside core only when native per-action assignment is required. Process 8 exposes that boundary but has a viable workaround. |
| `rules` | Do not add for these examples. Invoice thresholds are expressible in instructions; deterministic server enforcement is a stronger requirement. |
| `subprocess` | Defer. None of the ten needs it. |
| `evaluation` | Defer. None needs model-graded acceptance. |
| `actions` | Defer. The agents use their own tools; human tasks can handle payments and signatures. |

I would remove the sentence **“These five rules are the protocol”** as the conformance definition. It understates the document’s own required tools, wire behavior, and other MUSTs. A server can preserve those five broad invariants and still be unusable by another implementation’s client.

## Task 4 — Overall judgment

An organization could use this as a compact authoring convention for many sequential processes, especially where people perform semantic checks and existing systems own documents, payments, and action tracking. It could not yet adopt the draft as a portable production protocol: routing arguments contradict the tool signatures, human assignment and attachment access are incomplete, rejection leaves the validity of prior work unclear, event delivery is unspecified, and the publication hash excludes behavior-changing content. Eight examples fit the stated authoring model, one needs an external tracking workaround, and one genuinely requires concurrent human work. An agent that has never seen the document might operate a particular server if that server supplies rich MCP schemas and claim instructions, but **the specified tool results alone do not guarantee it can discover inputs, submit a route, retrieve evidence, interpret rejection data, or recover correctly**. Fix those contracts before adding more workflow features.

**Verification:** all ten files parse as YAML frontmatter, have unique step IDs and valid targets, and contain no cycles through ordinary continuation/timeout edges. No server execution or protocol-conformance test was performed.