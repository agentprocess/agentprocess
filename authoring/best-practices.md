---
title: "Best practices for process authors"
sidebarTitle: "Best practices"
description: "How to write processes that agents and people can act on without asking, and that produce the result they promise."
---

A valid file is easy to write. A process that delivers its result is harder. These practices are about the second. None of them adds a requirement to the format.

## Start from the result and the real work

Before writing steps, establish:

- **The result and who receives it.** What state does the recipient need at the end, and where will it be checked? "The supplier is approved for setup" and "the supplier is set up" are different results.
- **The boundary.** What starts the process, what must be true before it starts, where it ends, and what it hands to other processes.
- **Who owns it.** The person accountable for the process, and the people and roles who do and accept the work.
- **The facts it runs on.** The systems, records, tools, permissions and deadlines it depends on. Name the external systems in `requires.systems`, so an organization adopting the process knows what to connect first. Declaring a system grants no access to it.

Ask the people who do the work how a recent normal case and a recent difficult case went. Do not write down an imagined process and present it as the real one. Keep confirmed requirements apart from proposals and open questions, and do not invent thresholds, approvers or policy to make a draft look complete.

## Choose step boundaries deliberately

Start a new step when responsibility changes, when a result must be recorded, when someone decides, or when the process must wait. Do not turn every sentence into a step.

| Kind | Use it for |
|---|---|
| `agent` | Work an authorized agent can do with its own tools. |
| `task` | A person's work, including a person choosing between business routes. |
| `approve` | A real human approval. Say what the approver must inspect and when to reject. Do not add a signature without a purpose. |
| `wait`, `wait_until`, `wait_for` | A fixed delay, a date in the run's data, or an event from outside. |
| `finish` | The end. Name the outcome after the state actually reached. |

## Make each step a contract

For every agent and task step, the instructions, fields and evidence together should answer:

| Question | Where it goes |
|---|---|
| Who can do this, with which tools and access? | The step kind, the role, and the instructions. A role name grants no permissions. |
| What must be read, and what if it is missing or contradictory? | The instructions. |
| What must change or be produced, and what is out of scope? | The instructions. |
| Which results do later steps or people need? | `output`, with units, identifiers and dates where they matter. |
| What shows it was done? | `evidence`, and what the summary should say. |
| What happens next, including on doubt, failure or rejection? | `next` with `when` labels, `on_reject`, and the instructions. |

Trace every later use of a field back to an input or an earlier output on every route that reaches it. The whole run's data is visible to every step, but a step that was skipped produced nothing.

## Write instructions people and agents can follow

- Start with a verb. One action per sentence. Use the same names for the same things.
- Put the condition before the action: "If the order is missing, escalate before continuing."
- Use **must** for requirements, **may** for permission and **do not** for prohibitions. Avoid "handle appropriately".
- State amounts, units, time zones and deadlines when they are confirmed. Flag them as missing when they are not.

Replace "Validate the request" with "Compare the requested items with the approved order. Record each mismatch. If the order is missing, escalate." Declare the mismatches as an output if a later step needs them.

Keep each rule in the step that applies it. Use the body for shared context and policy, and do not write two versions of the same rule.

## Routes

When a step can go more than one way, give `next` as a list and label each route with `when`. The actor chooses and records a reason. Write `when` texts that actually distinguish the outcomes, and say in the instructions what to do when none fits or the facts are missing. Do not force an arbitrary choice.

There are no rule tables. If a wrong choice would be costly, put a person on the decision with a `task`, or add a `check` on the output the choice depends on.

## Know what each control proves

| Control | What it establishes | What it does not |
|---|---|---|
| Field and route rules | Declared types, required fields, allowed routes and evidence kinds. | That the content is true or within policy. |
| `file` evidence | The file exists and its bytes match the hash recorded at upload. | That its contents are right. A `link` is only a recorded reference. |
| A calculation or system read-back | An objective fact, when the actor has a tool that can do it. Name the tool. | Anything, if the server is expected to run it. It does not. |
| A `check` | A model's judgment of the material it was shown. | Anything in a file it cannot open, or a calculation. |
| An approval | A recorded human decision. | Authority or separation of duties. Those are established by your organization, outside the protocol. |

## Plan for things going wrong

- **Missing input, unavailable system or unclear policy:** say what is needed and send it to a person with `escalate` rather than inventing success.
- **An external action with an uncertain result:** check the target system before trying again, and name the identifier used to find an earlier attempt. The protocol does not undo anything a run did outside the server.
- **Rejection and rework:** say which work repeats, which evidence must be refreshed, and which external actions must be reused rather than repeated. An `on_reject` loop has no limit of its own; say when repeated failure should go to a person.
- **Lateness:** `due` marks work overdue and notifies people. It is not a business calendar and it does not fail the step. A `wait_for` can take `timeout` and `on_timeout`.

## Earn trust before removing people

Start with approvals where trust has not been earned. Then add a `check` with `advisory: true` to the step before the approval, and compare its verdicts with the approvers' decisions on real cases. Agree how many false passes and false fails are acceptable before you let the check block or replace an approval. Time passing is not evidence. Keep approvals that policy requires.

## Keep data where it belongs

Everyone who works a step sees the whole run's data. Keep identity documents, bank details and secrets in the systems that own them, and put references in the run.

If a process grows so large that whole-run visibility becomes a problem, it is two processes.

## Use the body for what the YAML cannot say

After the frontmatter, a short body helps everyone. Use these sections when they add something, and merge or drop them when they would repeat each other:

```markdown
# Process title

## Purpose and completion
Who receives the result, what it is, and what proves each outcome.

## Start and scope
What starts it, what must be true first, what is excluded, and the hand-offs.

## Ownership and resources
The accountable owner, who does what, the access needed, and policy references.

## Exceptions and recovery
Escalation, overdue work, and recovery from external actions.

## Measures and review
What is measured, from where, who reviews it and when.
```

Do not invent YAML keys for ownership, scope or measures; unknown keys are refused. They belong in the body.

## Check before calling it ready

1. **Validate the file** on a server, or with a parser, and confirm every role maps to a real group.
2. **Walk representative cases:** a normal case, each different outcome, and the failure and rework paths. Use boundary values for thresholds.
3. **Run it in test mode** with someone other than the author following it. A test run does not prove that live integrations work.
4. **Check the outcome** against the recipient's criteria and the system of record. Reaching a `finish` is not the same as achieving the result.

Once it is live, agree one outcome measure and one flow measure with the owner, review errors, delays and disagreements, and publish a new version when the cause is fixed. Running work keeps the version it started on.
