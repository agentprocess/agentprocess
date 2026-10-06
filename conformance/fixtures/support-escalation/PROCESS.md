---
name: support-escalation
description: Resolve a customer support case at tier 1 or pass a reproducible problem to engineering and confirm the resolution.
inputs:
  ticketId: string
steps:
  - id: diagnose
    agent: |
      Read the ticket, permitted diagnostic logs and known fixes. Remove secrets
      from anything attached. Apply only fixes authorized by the support runbook.
      Choose confirm if fixed; otherwise choose tier1_handoff.
    output:
      diagnosis: string
      attemptedFixes: string
    evidence: [link]
    next: [confirm, tier1_handoff]
  - id: tier1_handoff
    person: tier1-lead
    task: |
      Check reproduction steps and business impact. Add missing diagnostics and
      customer contact details to the existing support ticket. Do not open a duplicate.
    output:
      reproduction: string
      impact: string
      severity: { type: string, one_of: [low, high, critical] }
    evidence: [link]
    due: 4h
  - id: engineering
    person: engineering-oncall
    task: |
      Own the issue through a deployed fix or an agreed workaround. Record the
      issue and deployment references in the support ticket. Keep the task open
      while investigating; do not mark a referral as a completed resolution.
    output:
      resolution: string
      issue: string
    evidence: [link]
    due: 1d
  - id: confirm
    person: tier1-lead
    approve: Confirm with the customer that the reported problem is resolved and that the ticket contains the resolution.
    on_reject: tier1_handoff
    due: 2d
  - id: close_ticket
    agent: Close the existing support ticket with the confirmed resolution. If already closed, return its current reference.
    output:
      ticket: string
    evidence: [link]
  - id: done
    finish: resolved
---

# Customer support escalation

Start from an open support ticket. Engineering work can take several days;
the engineering task remains open. Tier 1 owns customer communication.
If confirmation fails, the case returns through tier 1 to engineering.
