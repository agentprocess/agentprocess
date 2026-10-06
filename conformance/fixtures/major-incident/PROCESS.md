---
name: major-incident
description: Open an incident bridge, have operations, security and communications respond at the same time, and get the commander's closure.
inputs:
  incidentId: string
  commander: person
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
---

# Major incident response

The agent opens the bridge and pages the leads. Operations, security and
communications each get their own work item at once and work concurrently.
The commander closes the incident only after all three finish; a rejection
reopens every branch with the earlier work kept in history.
