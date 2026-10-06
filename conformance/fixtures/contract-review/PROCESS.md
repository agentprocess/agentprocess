---
name: contract-review
description: Obtain legal and finance approval of the same contract revision before an authorized person signs it.
inputs:
  contractId: string
  document: string
  owner: person
steps:
  - id: prepare
    person: inputs.owner
    task: |
      Prepare an immutable contract revision in the document system. Resolve
      any rejection note. Attach that exact revision, not a latest-document URL.
      Summarize changes and commercial terms, including currency and liability cap.
    output:
      revision: string
      summary: string
    evidence: [file]
  - id: legal_review
    person: legal-reviewer
    approve: Approve the exact prepare.revision and attached file for legal terms. Reject with required changes if unacceptable.
    on_reject: prepare
    due: 3d
  - id: finance_review
    person: finance-reviewer
    approve: Approve the same prepare.revision for pricing, tax, payment terms and budget. Any requested edit requires rejection.
    on_reject: prepare
    due: 2d
  - id: sign
    person: authorized-signatory
    task: |
      Sign only the exact revision approved by legal and finance. Verify its
      document-system revision against prepare.revision before signing.
      If a new edit is needed, fail this run with a note and start a new review.
      Attach the executed agreement.
    output:
      signedRevision: string
      signedAt: datetime
    evidence: [file]
    due: 2d
  - id: done
    finish: signed
---

# Contract review

The contract owner starts this before signature. Legal then finance review
the same document. A rejection returns to preparation and repeats both reviews.
An older approval never authorizes a revised document. This run records the
signature; the authorized signatory uses the company's signature service.
