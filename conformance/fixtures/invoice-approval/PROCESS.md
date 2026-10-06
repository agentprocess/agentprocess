---
name: invoice-approval
description: Validate a supplier invoice and obtain the approval required for its total before recording it as approved for payment.
inputs:
  invoiceId: string
  invoiceDocument: string
  manager: person
steps:
  - id: validate
    agent: |
      Read the invoice and match supplier, purchase order and goods receipt in ERP.
      Check for duplicates. Use the invoice total including tax, in USD.
      If it is not in USD, escalate for a finance-approved USD valuation.
      Escalate discrepancies; do not route an invalid invoice for approval.
      Attach the invoice and match report as links.
      Choose manager_review for totals up to and including 10000;
      choose finance_review for totals above 10000 up to and including 100000;
      choose cfo_review for totals above 100000. Explain the amount and choice.
      These are replacement approval tiers, not cumulative sign-offs.
    output:
      totalUsd: number
      supplier: string
      matchSummary: string
    evidence: [link]
    next: [manager_review, finance_review, cfo_review]
  - id: manager_review
    person: inputs.manager
    approve: Approve the matched invoice only if totalUsd is at most 10000 and the purchase is justified.
    due: 2d
    next: record
  - id: finance_review
    person: finance-controller
    approve: Approve only if totalUsd is above 10000 and at most 100000, the match is valid and funds are available.
    due: 2d
    next: record
  - id: cfo_review
    person: cfo
    approve: Approve only if totalUsd is above 100000, the match is valid and the expenditure is justified.
    due: 2d
    next: record
  - id: record
    agent: |
      Record approval in ERP against invoiceId with the approving person's identity.
      Check the existing approval first so a retry does not create a duplicate.
      This process authorizes payment; it does not execute payment.
    output:
      approvalRecord: string
    evidence: [link]
  - id: done
    finish: approved
---

# Invoice approval

Accounts payable starts one run per invoice. Finance maps the controller and
CFO roles before use. A rejection ends this request; a corrected invoice gets
a new request. Never change the amount after approval.
