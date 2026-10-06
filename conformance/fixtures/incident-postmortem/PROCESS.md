---
name: incident-postmortem
description: Review an incident and keep its corrective actions open until every action has verified closure evidence.
inputs:
  incidentId: string
  incidentOwner: person
steps:
  - id: draft
    agent: |
      Read the incident record and prepare a factual timeline, impact summary,
      contributing factors and proposed corrective actions. Each action needs
      a specific deliverable, one owner and a due date. Avoid blame.
    output:
      report: string
      proposedActions: { type: list, items: object }
    evidence: [link]
  - id: review
    person: inputs.incidentOwner
    approve: Approve the report and action scope only if every proposed action has an owner, due date and verifiable completion criterion.
    on_reject: draft
    due: 5d
  - id: register_actions
    agent: |
      Create or find one issue per approved action in the existing issue tracker,
      linked to incidentId. Set its owner and due date. Do not duplicate issues
      when retried. Attach a tracker view containing every action.
    output:
      actionIds: { type: list, items: string }
      trackerView: string
    evidence: [link]
  - id: track_closure
    person: inputs.incidentOwner
    task: |
      Track every action in the issue tracker. Chase owners and overdue work there.
      Keep this task open until every action has closure evidence meeting its
      approved completion criterion. Reopen inadequate fixes in the tracker.
      Return one closure entry per actionId, with issue ID and evidence link.
      If there are no approved actions, state why none were needed.
    output:
      closures: { type: list, items: object }
      closureSummary: string
    evidence: [link]
    due: 30d
  - id: done
    finish: closed
---

# Incident postmortem

Start after service recovery. Approval of the report is not closure of the
actions. The incident owner remains accountable until every action closes.
The issue tracker owns per-action assignments, due dates and progress; this
run has one aggregate closure task. An overdue task remains open.
