---
name: expense-reimbursement
description: Check an employee's expense receipts, obtain their manager's approval and reimburse the approved amount.
inputs:
  claimId: string
  manager: person
steps:
  - id: receipts
    person: initiator
    task: |
      Attach readable receipts for every expense line and explain the business
      purpose. Use USD amounts; attach the approved exchange-rate record for
      foreign-currency receipts. Correct any rejection note.
    output:
      purpose: string
      totalUsd: number
      lines: { type: list, items: object }
    evidence: [file]
  - id: check
    agent: |
      Read every receipt and match it to one expense line. Each line must state
      merchant, date, amount and business purpose. Recalculate totalUsd, check
      expense policy in the expense system and search for duplicate claims.
      Escalate missing receipts, mismatches or exceptions before submitting.
    output:
      checkedTotalUsd: number
      checks: string
    evidence: [link]
  - id: manager_review
    person: inputs.manager
    approve: Approve the checked amount only after reviewing the business purpose and receipts. Do not approve your own expenses.
    on_reject: receipts
    due: 3d
  - id: reimburse
    person: accounts-payable
    task: |
      Verify manager approval and check claimId for an existing payment.
      Pay the approved amount using the employee record in the expense system.
      Do not put bank details in this run. Return the settled payment reference.
    output:
      paymentReference: string
      paidUsd: number
    evidence: [link]
  - id: done
    finish: reimbursed
---

# Expense reimbursement

The employee starts one run per expense claim. Each receipt must be attached;
one attachment does not excuse missing receipts on other lines. On rejection,
resubmit receipts and repeat the checks before another manager decision.
