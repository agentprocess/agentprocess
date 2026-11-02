# Agent Process

Design shared business processes with Claude and carry out the agent steps your organization has authorized, with evidence on every step and human approvals where the process asks for them.

## Use it

1. Install the plugin.
2. On the plugin's **Connectors** tab, connect the Agent Process connector and sign in to your organization's Agent Process workspace.
3. Ask Claude to list the published processes in your workspace, draft a process with a human approval step, or pick up the agent work you are authorized to do.

Claude carries out an assigned step with its own tools and reports the result back through the connector. People make approval decisions in the Agent Process web app.

## What it covers

- **Processes**: draft in chat, see the map, publish, import and export the PROCESS.md file. Steps carry outputs, evidence, routes, parallel branches, rejection loops and AI checks (Jev).
- **Runs**: start, follow, claim and complete agent steps, escalate, and inspect activity and recovery.
- **Approvals**: approval steps on a frozen evidence snapshot, decided in the app or with one tap from email. Claude never decides.
- **Data sources**: PostgreSQL, MongoDB, SQL Server, Cosmos DB, or a REST API from OpenAPI, Swagger or Postman, with reads you choose and writes an admin approves. Credentials stay in the vault.
- **Emails in your words**: templates with fill-in fields and conditional sections, sent by Claude during a run within the recipient rules you set.
- **Publish for others**: a key for another team, client or partner, limited to chosen processes.
- **The record**: actor, channel, reads and writes with receipts, emails, check results and decisions, kept with every run.

## Try asking

- "Draft our refund process: the agent reads the order and assesses the claim, Finance approves anything over $500, then it issues the refund and emails the customer."
- "Where is the refund for order #5521, and who is it waiting on?"
- "Find the agent work I am authorized to pick up."

## Data

Process definitions, run data, and your sign-in go to your organization's Agent Process workspace at api.agentprocess.io. The plugin stores nothing itself.
