---
name: major-incident-response
description: Coordinate operations recovery, security containment and customer communication during a major incident.
inputs:
  incidentId: string
  commander: person
steps:
  - id: activate
    agent: |
      Open or find the incident bridge and page operations, security and customer
      communications together. Tell all three to start immediately and record
      their progress in the incident system. Do not wait for service recovery
      before notifying security or communications.
    output:
      bridge: string
      pagingReceipt: string
    evidence: [link]
  - id: operations
    person: operations-lead
    task: Restore service, verify health checks and attach the incident timeline and recovery evidence.
    output:
      recoverySummary: string
    evidence: [link]
    due: 1h
  - id: security
    person: security-lead
    task: Contain the security impact, preserve relevant evidence and record remaining exposure.
    output:
      containmentSummary: string
    evidence: [link]
    due: 1h
  - id: communications
    person: communications-lead
    task: Publish the initial customer notice and updates in the incident system. Attach the communication record.
    output:
      communicationsSummary: string
    evidence: [link]
    due: 1h
  - id: close_review
    person: inputs.commander
    approve: Confirm that operations, security and communications have each independently completed and supplied evidence before closing.
  - id: done
    finish: stabilized
---

# Major incident response

All three leads must receive independent process work items at activation,
work concurrently and submit evidence immediately, even if another lead is
still working. Their one-hour deadlines run from activation. The commander
must see their separate live statuses and close only after all three finish.
Paging outside the process is a fallback, not fulfillment of those requirements.
This section-2-only attempt records the work sequentially and therefore does
not implement the required live coordination.
