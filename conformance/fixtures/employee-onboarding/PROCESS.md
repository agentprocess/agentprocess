---
name: employee-onboarding
description: Prepare a new employee's HR record, equipment and first-week plan before their start date.
inputs:
  employeeId: string
  employeeName: string
  startDate: date
  hiringManager: person
steps:
  - id: hr_setup
    person: hr-operations
    task: |
      Confirm the signed employment agreement and required checks in HRIS.
      Create or confirm the employee record. Return only the HRIS reference,
      work location and approved job title; do not attach identity documents.
    output:
      hrisRecord: string
      location: string
      jobTitle: string
    evidence: [link]
    due: 2d
  - id: access_plan
    person: inputs.hiringManager
    task: Specify the applications, access levels and equipment needed for this job. Name the first-week buddy.
    output:
      accessRequests: { type: list, items: string }
      equipment: string
      buddy: person
    due: 2d
  - id: it_setup
    person: it-operations
    task: |
      Provision the approved accounts and equipment. Keep accounts disabled until
      the start date. Attach the asset and access-ticket references, never passwords.
      Resolve missing approvals before completing this task.
    output:
      assetTag: string
      accounts: { type: list, items: string }
      handoverPlan: string
    evidence: [link]
    due: 3d
  - id: welcome
    agent: |
      Create a first-week checklist using HR, IT and manager outputs. Send the
      manager the equipment handover plan and account activation checklist.
      Check employeeId for an existing checklist before creating one.
    output:
      checklist: string
    evidence: [link]
  - id: confirm_ready
    person: inputs.hiringManager
    approve: Confirm that equipment, access activation, buddy and first-week meetings are arranged before startDate.
    due: 1d
    on_reject: access_plan
  - id: done
    finish: ready
---

# Employee onboarding

HR starts the run when the offer is accepted. This process covers preparation;
the manager uses the checklist on the first day. HR and IT work in sequence.
On rejection, update the access plan and repeat IT setup and the checklist;
reuse existing accounts and assets. All participants can see this run.
