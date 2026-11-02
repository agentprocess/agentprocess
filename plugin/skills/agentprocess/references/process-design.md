# Process design and assurance

Read when creating, reviewing or improving a PROCESS.md. These are authoring practices, not additional protocol requirements. Scale them to the process's risk and the requested change. A wording correction does not require a new discovery exercise or pilot.

The deliverable is an executable process whose completed work supports its stated outcome. A valid graph, accepted submission or model verdict alone does not establish that outcome.

## Establish the result and the real work

Use the request, existing process, policy documents and actual cases first. For a new process, establish:

- **Result and recipient:** who needs the result, the observable state they need, and where it will be verified.
- **Boundary:** trigger, entry requirements, start and end, exclusions, and handoffs to other processes.
- **Ownership:** the accountable process owner and the people or roles performing and accepting the work.
- **Operating facts:** authoritative information, tools, permissions, dependencies and timing constraints.

Ask practitioners how a recent normal case and a difficult case were handled. If they or their records are unavailable, identify that limitation. Do not present an imagined current workflow as observed fact.

Separate confirmed requirements, proposed choices and unresolved questions. Ask for missing information that changes authority, routing, acceptance or external effects. Continue drafting independent parts, but do not invent thresholds, approvers, service targets or organizational policy. A draft with unresolved requirements is not ready for operational use.

For an existing process, identify the observed problem and its cause before editing. Retain useful controls; remove steps or handoffs that add neither value nor a necessary control. Consider waiting time, workload and available capacity, not just the number of steps. Parallelize only independent work and only when the server supports it.

## Turn work into step contracts

Choose a step boundary when responsibility changes, a meaningful result must be recorded, a decision occurs, or the process must wait. Do not turn every sentence into a workflow node.

For each agent or person task, make these facts explicit where relevant:

| Part | What the author must resolve |
|---|---|
| Actor and access | Who can perform the work, with which available tools and permissions? A role label alone does not grant authority. |
| Inputs | Which run fields, records, policy versions or files must be read? What happens if they are missing or contradictory? |
| Action | What must change or be produced, and what is outside this step's scope? |
| Output | Which declared fields carry the result into later work? Include needed units, identifiers and dates. |
| Acceptance | What observable criteria establish completion, and who or what verifies them? |
| Evidence | Which record or artifact supports each material claim, and can its reviewer access it? |
| Continuation | When does each route apply? What happens on uncertainty, failure or rejection? |

Trace every downstream dependency back to an input or an earlier output on every route that reaches it. Whole-run visibility does not make a skipped step's output exist. Keep sensitive documents and secrets in their owning systems; use references because run data is shared with participants.

Use `agent` for work an authorized agent can do, `task` for a person's work, and `approve` for an actual human approval. Use `task` when a person must choose among business routes. For every approval, say what the approver must inspect and when to reject; do not add a signature without a purpose.

Write route conditions that distinguish the available outcomes. Explain how to handle missing facts, overlapping conditions or no matching condition; do not force an arbitrary choice. Name `finish` outcomes after the state actually achieved. For example, screening and approval can justify `approved_for_setup`; `onboarded` needs the setup and verification work too.

Check that the available protocol can express the required behavior. Core permits backward edges only through approval rejection; parallel-1 waits for every branch and has no global business-event interrupt. If a workaround adds a human gate, changes cancellation behavior or moves work outside the run, state that tradeoff and leave material business decisions unresolved until confirmed. Do not silently weaken the requirement to obtain a valid graph.

## Write instructions participants can follow

Use the requested language. Apply the repository's practical controlled-language style:

- Start procedural sentences with direct verbs. Use active voice, one main action per numbered instruction, and consistent business terms and field names.
- Put a condition before its action. State amounts, units, time zones and deadlines when confirmed; flag missing values instead of guessing them.
- Use **must** for requirements, **may** for permission and **Do not** for prohibitions. Avoid vague phrases such as “handle appropriately.”
- Separate actions from explanations. Preserve identifiers, source quotations, exceptions and authority when editing.
- For English, aim for 20 words per procedural sentence and 25 per descriptive sentence. Meaning and required conditions take priority over the target.

For example, replace “Validate the request” with “Compare the requested items with the approved order. Record each mismatch. If the order is missing, escalate before proceeding.” Declare the mismatch output if later steps need it.

Keep operational criteria in the relevant step. Use the body for shared context and policy; do not create competing versions of the same rule.

## Design controls and recovery that actually work

Distinguish these mechanisms when deciding how a requirement will be checked:

| Mechanism | What it establishes and what it does not |
|---|---|
| Core validation | Declared types, required fields, allowed route and evidence kinds. It does not establish truth or business-policy compliance. |
| File evidence | The uploaded artifact exists and its bytes match the recorded hash. Its contents can still be wrong. Core link evidence is only a recorded reference. |
| Deterministic verification | A calculation or system read-back can establish an objective fact. Name the tool or script the actor can actually use; the server does not execute it for them. |
| Model `check` | A judgment about the supplied material. The evaluator cannot open files; put relevant permitted content in output or run data, or use an authorized reviewer who can inspect the source. |
| Human approval | A recorded human decision. Required business authority or separation of duties must be established outside the core; a human identity alone does not prove either. |

Use separate check questions for distinct criteria. Do not ask a model to verify an inaccessible document or substitute a confidence score for a deterministic calculation. For a new check, use advisory mode where supported and compare its judgments with reviewed cases. Agree acceptable false-pass and false-fail behavior before enforcement; elapsed time or a high confidence setting is not validation. Keep policy-required approvals.

For relevant failure modes, specify a recovery owner and the next safe action:

- Missing input, unavailable service or unclear policy: identify what is needed and route to a person rather than inventing success.
- External action with an uncertain result: inspect the target system before retrying. Name the business identifier used to find an existing result. Specify repair or compensation when a partial action warrants it; the protocol does not undo external effects.
- Rejection and rework: say which work repeats, which earlier evidence must be refreshed, and which external actions must be reused rather than repeated. An `on_reject` loop has no core retry limit; state when repeated failure should go to an operator.
- Lateness or a missing event: `due` is elapsed time from step readiness and only marks overdue work and notifies. It is not a business calendar, completion guarantee or automatic failure route. A `wait_for` can use `timeout` and `on_timeout`; otherwise identify the required operational follow-up.

## Document the process without extending the schema

After the YAML, use this compact body structure when helpful. Merge sections that would merely repeat each other; omit inapplicable material. Replace prompts with confirmed facts or explicitly unresolved items.

```markdown
# Process title

## Purpose and completion
Recipient, intended result, and observable proof for each terminal outcome.

## Start and scope
Trigger, entry requirements, exclusions, and upstream/downstream handoffs.

## Ownership and resources
Accountable owner, role responsibilities, required access, and policy references.

## Exceptions and recovery
Shared escalation, overdue-work and external-action recovery instructions.

## Measures and review
Outcome and flow measures, their data sources, review owner and review triggers.
```

Keep policy/reference files in the package only when supported by the target server's profiles. Follow its advertised authoring tools for bindings and publication; those operations are not core protocol tools. Drafting does not authorize publishing, starting a live run or changing access. Use existing user authorization without requesting it again.

## Verify readiness and improve from evidence

Before declaring a new or materially changed process ready:

1. **Check the definition.** Use the available parser or server validation. Verify supported profiles, role/resource bindings, reachable paths, declared field references and agreement between prose and YAML. If a server is unavailable, report structural checks separately from unverified bindings and capabilities.
2. **Walk representative cases.** Trace a normal case, each materially different outcome, and relevant failure/rework paths. Use boundary values for policy thresholds. At each transition, identify the available inputs, action, evidence, decision and expected business state. Select cases by risk, not by a fixed test count.
3. **Try the procedure.** Have someone other than the author follow it with representative data where practical. In authorized test runs, keep external effects simulated or captured, including actions performed with the agent's own tools. Compare results against predetermined expectations. A walkthrough is not a test run, and a test run does not prove live integrations work.
4. **Verify the outcome.** Check the recipient's acceptance criteria and the relevant system of record. Report what was actually tested, what failed and what remains unverified. Do not equate reaching `finish` with achieving the business result.

For a review, return findings tied to the step, a concrete failure scenario, its consequence and the smallest correction. For authoring, deliver the PROCESS.md, unresolved decisions and verification status. Do not create extra reports unless requested or useful for a material gap.

For an authorized pilot, agree a small set of measures with the owner: an outcome measure such as accepted results without rework, and a flow measure such as end-to-end elapsed time. Specify the data source, measurement window and target or baseline; do not invent targets. Review errors, delays and reviewer disagreements, correct their causes, then publish a new version when authorized. Existing runs retain their pinned version. Review again when policy, dependencies or measured results change.

## Basis

These sources inform the authoring method; they are not runtime dependencies or claims of certification:

- [ISO process approach](https://www.iso.org/files/live/sites/isoorg/files/archive/pdf/en/iso9001-2015-process-appr.pdf): intended outcomes, ownership, interactions, risks, measurement and improvement.
- [EPA SOP guidance](https://nepis.epa.gov/Exe/ZyPURL.cgi?Dockey=P1008GTX.txt): practitioner input, reproducible instructions and validation by another participant.
- [Lean standardized work](https://www.lean.org/lexicon-terms/standardized-work/): actual sequence and capacity as a baseline for improvement.
