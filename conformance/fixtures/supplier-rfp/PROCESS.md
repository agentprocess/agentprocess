---
name: supplier-rfp
description: Collect three comparable supplier quotes and select one with procurement approval.
inputs:
  rfpId: string
  requirements: string
  supplierOne: string
  supplierTwo: string
  supplierThree: string
steps:
  - id: collect_quotes
    person: procurement-buyer
    task: |
      Send the same requirements and requested deadline to the three named
      suppliers. Obtain one current written quote from each, with price,
      currency, scope, delivery date and validity date. Attach all three quotes.
      Keep this task open until all arrive. If a supplier declines, fail this
      run with a note; start a new request with a replacement supplier.
    output:
      quoteOne: string
      quoteTwo: string
      quoteThree: string
    evidence: [file]
    due: 7d
  - id: compare
    agent: |
      Compare all three attached quotes against the same requirements. Flag
      exclusions and normalize costs with stated currency assumptions. Escalate
      missing or incomparable quotes. Recommend exactly one named supplier and
      explain price, delivery and fit; do not place an order.
    output:
      comparison: string
      recommendedSupplier: string
      rationale: string
  - id: select
    person: procurement-manager
    approve: Approve the recommended supplier and comparison. Reject if the quotes or selection need revision.
    on_reject: collect_quotes
    due: 2d
  - id: record_selection
    agent: |
      Record the approved supplier and quote in the sourcing system against
      rfpId, checking first for an existing selection. Notify the buyer there.
    output:
      sourcingRecord: string
    evidence: [link]
  - id: done
    finish: selected
---

# Supplier RFP

The buyer starts with three distinct suppliers. Procurement owns collection
and follow-up; this process records the three quotes and the selection. An
order requires the company's separate purchasing authorization.
