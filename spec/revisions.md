---
title: "Revisions"
description: "Every revision of the specification and what prompted it."
---

Changes to the specification, newest first. The wire version is `core-2`; revisions are editorial and contract refinements within it, each made after a review or after building the reference server. Profiles carry their own versions (`parallel-1`, `check-1`).

## Revision 9

Source: publishing the specification.

Editorial. §2 no longer says work items carry the body, which revision 8 removed; the body comes with every claim and every run view. The tools document states the optional `extensions` on a claim result, which the JSON Schema already had.

## Revision 8

Source: independent reviews of the reference server.

Work items carry no body or run data (the tools file was right); the content hash separator is one newline; the run-data cap is normative and bounds inputs and held events; schema violations are `invalid`; `send_event` is for agents and operators; the name pattern is exact; dates must be real.

## Revision 7

A `next` route may carry a `when` label, so a process map can show why each edge is taken without a rules engine.

## Revision 6

Source: building the reference server.

The returned note travels in `handoff`, which work items also carry; `already_decided` is checked before `stale`; `next` on a single-next step is a `not_accepted` issue; waits, finishes and failed server steps record who and what; paths must name declared fields; a token that does not verify is `invalid`.

## Revision 5

`summary` on every submission; `progress` on `renew`; `handoff` on `claim` and `get_work`; the `check` profile replaces the `evaluation` placeholder.

## Revision 4

Source: [independent review, round 3](https://github.com/agentprocess/agentprocess/blob/main/research/review-round-3.md).

`get_work` is the inbox for people too and work items carry their `decide` call; the rejecting approval keeps its note current and rewound steps show `null`; a `failed` decision leaves a `failed` step with its note and the run records `ended`; writes to an ended run return `conflict` with `run_ended`; a write returns only after the transitions it triggered are applied; an event with no wait is held until the run ends; `assignedTo` has a person form. The parallel step no longer records anything in run data, and the profile states what it changes instead of claiming nothing changes.

## Revision 3

Source: [independent review, round 2](https://github.com/agentprocess/agentprocess/blob/main/research/review-round-2.md).

Work items carry the step's output and evidence requirements; a replay always returns the original result; approvals no longer take a list `next`; held events are consumed oldest first; the snapshot is taken at read time so a stale decision can be retried; the `initiator` check covers every step; ending or cancelling a run cancels unfinished steps and tokens; human presence is stated as server authentication outside the protocol; `accepted` and `delivered` flags are dropped. The `parallel` profile is drafted against the 23 questions the review raised.

## Revision 2

Source: [independent review, round 1](https://github.com/agentprocess/agentprocess/blob/main/research/review-round-1.md).

The example's approval now reaches `done`; `submit` and `decide` carry `next` and `reason`; `upload` carries `requestId`; file evidence is openable; approvals record `decision`, `note`, `by`, `at`; rejection moves later work to history; events are held until their wait; the hash covers the body; `decide` is required; escalation goes to operators; `list_processes` shows `inputs`; `start_run` returns the run view. Removed: `x-` fields, `x-content-hash`, `overdue`, `withdrawn`, `retry`, `download` and `files/` (now the `files` profile), `examples/`, `version`, `license`, and keeping undeclared submitted fields.

## Revision 1

The first revision: one `PROCESS.md` with YAML frontmatter and a plain-language body, seven step kinds, ten required tools, seven error codes, and five server guarantees.
