---
name: vendor-check
description: Check a new supplier against sanctions lists, with a model check on the screening, and get finance approval.
inputs:
  vendorName: string
  amount: { type: number, description: Expected annual spend in USD }
requires:
  systems: [sanctions-screening]
steps:
  - id: check_vendor
    agent: |
      Check the vendor against the OFAC and EU sanctions lists.
      Attach the screening report. Choose approve when the vendor is
      cleared and decline when it is not.
    output:
      cleared: boolean
      summary: string
    evidence: [file]
    next: [approve, decline]
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

  - id: approve
    person: finance-approver
    approve: Approve only when the vendor is cleared and the spend is justified.
    due: 2d
    on_reject: check_vendor
    next: done

  - id: decline
    finish: declined

  - id: done
    finish: onboarded
---

# Vendor check

Start this when procurement has a new supplier. The agent screens the vendor
and a model checks the screening; finance approves, and the supplier is
onboarded. If finance rejects, the agent re-checks with the finance note.
