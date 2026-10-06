# Contributing

Thank you for helping. Agent Process stays useful by staying small, so this file says what is welcome and what is not.

## Where to post

- **Questions and proposals:** GitHub Discussions. Start here for any change to the specification.
- **Bugs:** issues. A bug is a contradiction, an ambiguity two implementations could read differently, a schema that disagrees with `spec/tools.md`, or a broken example.
- **Pull requests:** for fixes to wording, examples, schemas and broken links, and for proposals already agreed in Discussions.

## The bar for changes to the specification

Every new requirement is a cost to every implementation, and it is much easier to add to a specification than to remove from it. When in doubt, leave it out.

- A proposal starts from a real process. Show the `PROCESS.md` you tried to write and the point where the core could not express it, or the two behaviours two servers could legitimately choose. Theoretical completeness is not a reason.
- Prefer plain-language instructions to new keys. If an agent or person can do it from a sentence in the step, it does not need a key.
- A new feature arrives as a one-page **profile**, not as a change to the core, and only when a written process cannot be expressed without it.
- The tool contracts are the exception: gaps there are always worth fixing, because an agent can fill a gap in instructions but not in a contract.

Not accepted without a written process that needs them: decision tables, per-item fan-out, subprocesses, waivers, applicability rules, acceptance-recovery policies, classifiers and server-executed actions.

## Listing an implementation

Servers and agents are listed on the site when they are public, work today rather than in beta, and pass the conformance suite (servers) or complete the `get_work → claim → submit` loop against a conforming server (agents). Open a pull request with the entry and a light and a dark logo, SVG preferred.

## AI assistance

Say in your pull request or issue if AI tools helped write it, and how. For example: "Drafted with an AI assistant; I checked every example against the specification."

## Licenses

Contributions to the specification, schemas, skill and code are licensed under Apache-2.0.
