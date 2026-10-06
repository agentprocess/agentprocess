---
name: customer-offboarding
description: Authorize a customer's exit, cancel services, settle the final invoice and verify authorized data deletion.
inputs:
  customerId: string
  effectiveAt: datetime
steps:
  - id: authorize
    person: account-owner
    approve: Confirm the authenticated customer request, cancellation date and agreed export arrangements in the customer record.
    due: 2d
  - id: effective_time
    wait_until: inputs.effectiveAt
  - id: cancel_services
    agent: |
      Confirm the customer's export handover in the customer record. Stop
      renewals and deactivate billable services at the agreed time. Check each
      existing service status before writing and record each cancellation ID.
      Escalate an uncompleted export or a service that cannot be cancelled.
    output:
      cancellationIds: { type: list, items: string }
    evidence: [link]
  - id: final_invoice
    agent: |
      Reconcile usage, credits and deposits. Create or find the final invoice
      for customerId and this cancellation. Choose deletion_clearance if the
      balance is already settled or zero; otherwise choose settlement_event.
    output:
      invoiceId: string
      balance: number
      currency: string
    evidence: [link]
    next: [deletion_clearance, settlement_event]
  - id: settlement_event
    wait_for: final-invoice-settled
    timeout: 30d
    on_timeout: finance_followup
    next: verify_settlement
  - id: verify_settlement
    agent: |
      Read the billing ledger; do not treat the event as proof of payment.
      Choose deletion_clearance only if this final invoice is settled;
      otherwise choose finance_followup and explain the discrepancy.
    output:
      ledgerSummary: string
    evidence: [link]
    next: [deletion_clearance, finance_followup]
  - id: finance_followup
    person: finance-collections
    task: Resolve the final invoice through payment or an authorized write-off. Keep this task open until the ledger shows settlement.
    output:
      settlementReference: string
    evidence: [link]
    next: deletion_clearance
  - id: deletion_clearance
    person: data-steward
    task: |
      Confirm export completion and document the approved deletion scope,
      retention exceptions and any hold in the data inventory. Keep this task
      open until deletion is authorized. Do not remove retained records.
    output:
      deletionScope: string
      retentionExceptions: string
    evidence: [link]
  - id: delete_data
    agent: |
      Delete or schedule expiry only for the approved scope using the data
      inventory and approved deletion tools. Check existing deletion jobs before
      creating any. Return receipts identifying every system and any expiry date.
    output:
      deletionReceipts: { type: list, items: string }
    evidence: [link]
  - id: verify_deletion
    person: data-steward
    task: |
      Verify every system receipt and every scheduled expiry has completed.
      Keep this task open until all approved deletion is complete. Record
      retained exceptions and send the customer a truthful completion statement.
    output:
      completionStatement: string
    evidence: [link]
  - id: done
    finish: offboarded
---

# Customer offboarding

The account owner starts one run per cancellation. Billing sends
final-invoice-settled with this run ID after posting settlement. The agent
rechecks the ledger. This company settles its final balance before its routine
deletion workflow; separately mandated deletion deadlines are handled by the
data steward. Cancellation of this run does not restore cancelled services.
