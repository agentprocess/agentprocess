---
title: "Profile: check"
description: "A model answers fixed questions about each submission; a person sees only what it is unsure of."
---

Version `check-1`, 6 October 2026. Extends the [specification](../specification.md). The profile names no model or provider: any evaluator that can answer the three question kinds below (yes/no with a probability, a choice, a rubric level) conforms.

A server that implements this profile lists `check-1` in `describe.profiles` and the limits in §5, and lists it only while an evaluator is configured. A process uses it by putting a `check:` key on an `agent` or `task` step. A server without the profile, or without an evaluator configured, refuses such a process at publication.

## 1. Why

A process starts with approvals because nobody yet trusts the agent's work. A check is how it earns that trust back one step at a time: a model answers fixed questions about each submission, and a person is involved only when the model is unsure. Where policy still requires a signature, a check on the step before the approval pre-screens for the approver.

## 2. The key

Short form, one yes/no question:

```yaml
  - id: check_vendor
    agent: Check the vendor against the OFAC and EU lists. Attach the report.
    output: { cleared: boolean, summary: string }
    evidence: [file]
    check: The cleared verdict matches what the attached report says, and the report covers both lists.
```

Full form, a list of questions of three kinds:

```yaml
    check:
      - ask: The cleared verdict matches what the attached report says.
        pass: 0.9
        fail: 0.5
      - ask: What does the summary do?
        one_of:
          states: States the verdict and the lists checked.
          hedges: Avoids a verdict or qualifies it heavily.
          unclear: Cannot tell.
        pass: [states]
        unsure: [unclear]
      - ask: How complete is the screening?
        levels:
          - Neither list is clearly covered.
          - One list is covered.
          - Both lists are covered with the search terms stated.
        pass: 2
        confidence: 0.8
```

| Key | Kind | Meaning |
|---|---|---|
| `ask` | all | The question, in plain language. Required. |
| `pass` | yes/no | The yes-probability at or above which the answer passes. Default from `describe.limits.checkPass`. |
| `fail` | yes/no | Optional. Below it the answer fails; between `fail` and `pass` it is unsure. Without it, below `pass` fails. |
| `one_of` | choice | Map of answer name to a description. Two or more. Makes the question a choice. |
| `pass` | choice | List of answer names that pass. Required. |
| `unsure` | choice | Optional list of answer names that are unsure. Any other answer fails. |
| `levels` | rubric | Ordered list of 2–10 level descriptions, worst first. Makes the question a rubric. |
| `pass` | rubric | The level, 0-based, at or above which the answer passes. Required. |
| `confidence` | choice, rubric | Optional. Below this confidence the answer is unsure. |
| `advisory` | step key beside `check` | Optional, `true` to record verdicts without ever blocking. For earning trust before enforcing. Allowed only when `check` is present. |

A question has exactly one of `one_of`, `levels`, or neither. The short form is one yes/no question with default thresholds. Questions in one check are answered against the same material and cannot see each other's answers; split compound judgments into separate questions.

What the evaluator sees: the step's instructions, the body, the run data, the submission's output, summary and evidence references, and each question. It does not open files; a check about file contents needs the relevant content in output or run data. The evaluator does not write explanations; its answers are the probability, the chosen name, or the level, with a confidence where the kind has one.

## 3. Running

The check runs after the objective rules of core §6 rule 3 pass and before the step completes. The verdict is the worst question: unsure outranks fail, fail outranks pass.

| Verdict | Effect |
|---|---|
| pass | The step completes as usual. |
| fail | `not_accepted`. `issues` lists each failed question as `check: <ask> → <answer>`. For yes/no the answer is `no (p)` when `p` is below `pass` and `yes (p)` otherwise; for a choice it is the answer name and its confidence, `hedges (0.9)`; for a rubric it is `level <n>: <level text> (confidence)`. Example: `check: The cleared verdict matches the report → no (0.12)`. Run data does not change; the verdict and the issues go to `handoff` so the next attempt sees them. A person's task submission gets the same refusal. |
| unsure | The call returns `ok` with the run view: the submission is held as the step's candidate and the step is escalated exactly as `escalate` does, with the questions and answers in the work item's `escalation.check` and the candidate in `escalation.candidate`. The person decides `returned` with a note, `completed` with the candidate or their own output, or `failed`. A person completing an escalated step is not checked again; a person completing an ordinary task is. |
| error | The evaluator did not answer. The server retries once; then the agent gets `retry` and resubmits with the same `requestId`. A server MAY instead escalate after repeated errors. With `advisory: true` an error is recorded as `verdict: "error"` and the step completes. |

With `advisory: true`, every verdict is recorded and none blocks.

Pass, unsure and advisory verdicts are recorded in run data as `steps.<id>.check`; a blocking fail is recorded in `handoff.check` instead, since the step did not complete:

```json
{ "verdict": "unsure", "advisory": false, "model": "example-evaluator-1",
  "questions": [ { "ask": "The cleared verdict matches …", "kind": "yes_no", "answer": 0.62, "result": "unsure" },
                 { "ask": "What does the summary do?", "kind": "choice", "answer": "states", "confidence": 0.91, "result": "pass" } ],
  "at": "2026-10-06T09:04:20Z" }
```

An approver sees it before deciding. When the same attempt continues, after a `fail` or a `returned`, the previous check travels in `handoff.check` and the agent's `handoff.issues` carries the same lines. Rework through `on_reject` is a fresh attempt: the handoff starts empty and the earlier verdict is in the step's `history`. While an unsure verdict waits on a person, run data shows it as `steps.<id>.check` even if the step completed before.

## 4. Patterns

- **Replace an approval.** Where policy allows, delete the `approve` step and put `check` on the agent step. People now see only the unsure cases.
- **Pre-screen an approval.** Keep the `approve` step and put `check` on the step before it. The approver reads `steps.<id>.check` and spends time where the model was unsure.
- **Earn trust first.** Start with `advisory: true`, compare recorded verdicts against the approver's decisions for a few weeks, then remove `advisory`.
- **Route on a verdict.** A later step's instructions may tell the actor to read `steps.<id>.check` when choosing from its `next` list.

## 5. Server limits and configuration

`describe.limits` carries `checkPass` (default yes/no pass threshold, server default 0.9), `checkFail` (default none), `checkConfidence` (default 0.8), and `checkStateBytes` (how much run data the evaluator is shown, server default 64 KB; larger run data is truncated oldest step first and the truncation is recorded on the verdict). The model and provider are server configuration and never appear in the process file; `model` on the verdict records what answered.

## 6. What this profile does not do

It does not check arithmetic, counts, dates, schema or authorization; those are the core's objective rules or the process's own steps. It does not read files. It is not a security boundary: a submission is untrusted input to the evaluator as much as to anyone, and a process that needs a hard guarantee keeps a person on the step.
