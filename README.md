# Agent Process

An open protocol for business processes that AI agents and people work through together.

A process is one `PROCESS.md` file: a short YAML frontmatter that names the process, its inputs and its ordered steps, and a body in plain language. A **process server** runs it. It hands each step to an agent or a person in order, accepts their output against what the step declares, and keeps the record. Agents connect over MCP with fourteen tools that have fixed names and exact contracts.

```markdown
---
name: supplier-onboarding
description: Check a new supplier against sanctions lists and get finance approval.
inputs:
  vendorName: string
steps:
  - id: check_vendor
    agent: Check the vendor against the OFAC and EU sanctions lists. Attach the report.
    output: { cleared: boolean }
    evidence: [file]
    next:
      - { to: approve, when: Vendor is cleared on both lists }
      - { to: decline, when: Any sanctions hit }
  - id: approve
    person: finance-approver
    approve: Approve only when the vendor is cleared and the spend is justified.
    on_reject: check_vendor
    next: done
  - id: decline
    finish: declined
  - id: done
    finish: onboarded
---

# Supplier onboarding

The agent screens the vendor, finance approves, and the supplier is onboarded.
```

## Why

Agents can do the steps of real work. What they lack is the thing around the steps: the order, the hand-off to a person who must decide, a check that the output is what was asked for, and a record of who did what. Agent Process puts that in a server and keeps the process file small. A server enforces five things: order, one actor per step at a time, acceptance, people decide, and immutability. Everything scenario-specific stays in plain-language instructions that the agent or person adapts to.

## Contents

| Path | What it is |
|---|---|
| [`spec/specification.md`](spec/specification.md) | The specification: the file format, running, tools, evidence, and what a server guarantees. |
| [`spec/tools.md`](spec/tools.md) | The exact argument and result shape of every tool. Part of the specification. |
| [`spec/profiles/`](spec/profiles/) | Optional profiles: [`parallel`](spec/profiles/parallel.md) (fork and join) and [`check`](spec/profiles/check.md) (model checks on submissions). |
| [`spec/revisions.md`](spec/revisions.md) | Every revision and why it was made. |
| [`schemas/core-2/`](schemas/core-2/index.json) | JSON Schemas (draft 2020-12) for the document, the shared shapes and every tool. |
| [`skills/agentprocess/`](skills/agentprocess/SKILL.md) | An [Agent Skill](https://agentskills.io) that teaches an agent the loop and how to write a `PROCESS.md`. Works on any conforming server. |
| [`conformance/`](conformance/README.md) | Twelve conformance fixtures: ten processes written by independent reviewers, and one for each profile. |
| [`research/`](research/README.md) | The three independent reviews that shaped the specification. |

## Status

Draft, version `core-2`. One server implements it today. It will be called a standard only when a second, independent implementation passes the conformance suite.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). Proposals go to GitHub Discussions.

## License

Apache-2.0. See [LICENSE](LICENSE).
