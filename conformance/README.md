---
title: "Conformance"
description: "The rules a conforming server keeps, and the fixtures to test it with."
---

A server conforms when it keeps the five rules of the [specification](../spec/specification.md) §6, offers the required tools with the contracts in [tool contracts](../spec/tools.md), refuses what §2 says to refuse, and reports what it offers in `describe`.

## Fixtures

Each fixture is a folder with one `PROCESS.md`. The first ten were written by independent reviewers ([research](../research/README.md)) and use the core only. A conforming server publishes all ten. The last two exercise the profiles; a server publishes each one only if it offers that profile.

| Process | What it exercises |
|---|---|
| [blog-publication](fixtures/blog-publication/PROCESS.md) | `wait_until` on an input date, approval with `on_reject` |
| [contract-review](fixtures/contract-review/PROCESS.md) | A person task, two approvals, a `person` path, file evidence |
| [customer-offboarding](fixtures/customer-offboarding/PROCESS.md) | Route lists, `wait_until`, `wait_for` with `timeout` |
| [employee-onboarding](fixtures/employee-onboarding/PROCESS.md) | `person` paths, approval with `on_reject` |
| [expense-reimbursement](fixtures/expense-reimbursement/PROCESS.md) | `initiator`, a `person` path, file evidence |
| [incident-postmortem](fixtures/incident-postmortem/PROCESS.md) | `person` paths, repeated rework through `on_reject` |
| [invoice-approval](fixtures/invoice-approval/PROCESS.md) | A route list, a `person` path |
| [major-incident-response](fixtures/major-incident-response/PROCESS.md) | Simultaneous work attempted in core only: the case that led to `parallel` |
| [supplier-rfp](fixtures/supplier-rfp/PROCESS.md) | File evidence, approval with `on_reject` |
| [support-escalation](fixtures/support-escalation/PROCESS.md) | A route list, approval with `on_reject` |
| [major-incident](fixtures/major-incident/PROCESS.md) | The `parallel` profile: fork, join, a `person` path |
| [vendor-check](fixtures/vendor-check/PROCESS.md) | The `check` profile on a route list with file evidence |

## Running it

Today the suite runs inside the reference server's own tests, which publish every fixture and replay the review traces over a real MCP client. A portable runner that takes any server's URL and a token is planned; until it exists, an implementer can use the fixtures and the traces in the review documents to test by hand.
