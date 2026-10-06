---
title: "The agentprocess skill"
sidebarTitle: "The skill"
description: "An Agent Skill that teaches any skills-compatible agent to work steps and write processes."
---

The `agentprocess` skill is an [Agent Skill](https://agentskills.io): a folder with a `SKILL.md` that a compatible agent loads when a task calls for it. It teaches the model the protocol, so you do not have to write that into your own prompts.

It covers:

- **Working steps:** the loop, how to read a claim (handoff first), how to submit, every error code and what to do about it, retries with `requestId`, and when to escalate.
- **Writing processes:** the `PROCESS.md` format, the rules a server checks, and a reference on process design and assurance for authors and reviewers.

It works on any conforming server. The connected server's own tool schemas and `describe` tell the agent what that server offers.

## Install it

The skill lives in the repository at [`skills/agentprocess`](../skills/agentprocess). Copy that folder into a skills directory your agent scans:

| Where | For |
|---|---|
| `.agents/skills/agentprocess/` in a project | Any client that follows the cross-client convention. |
| `~/.agents/skills/agentprocess/` | The same, for every project. |
| Your client's own directory, such as `.claude/skills/` | Clients with their own location. |

```bash
git clone https://github.com/agentprocess/agentprocess.git
mkdir -p .agents/skills
cp -r agentprocess/skills/agentprocess .agents/skills/
```

Then connect the agent to a process server's MCP endpoint. When a task involves process steps or a `PROCESS.md`, the agent loads the skill and follows it.

## Without skills support

An agent that does not support skills can still work steps. Follow [How to add Agent Process support](adding-support.md), or put the contents of `SKILL.md` in the agent's instructions.
