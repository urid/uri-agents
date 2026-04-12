# Email Triage Agent

Triages urid@crazylabs.com inbox using Claude Code + Gmail MCP + Slack MCP.

## Prerequisites

- Claude Code with google-workspace MCP configured (see Task 4 of implementation plan)
- Slack MCP already configured
- `email-agent/state.md` initialized with Slack channel ID and canvas ID

## How to Run

Start a Claude Code session and say:

```
Read email-agent/agent_prompt.md and run the email triage agent.
```

Claude will:
1. Fetch up to 50 unread emails
2. Triage each one against the rules in preferences.md
3. Archive noise, label with "agent", create Gmail drafts for answerable emails
4. Send Slack DM alerts for VIP/urgent emails
5. Update the Slack canvas with digest + action items
6. Update state.md with run stats

## Configuration

All behavior is controlled by `email-agent/preferences.md`. Edit that file to:
- Add/remove VIP senders
- Adjust archive rules
- Change urgency keywords
- Switch from draft-only to auto-send (when ready)

## After a Run

Check Slack DM to yourself:
- **DM messages** — VIP and urgent alerts
- **Canvas** — digest and action items

Emails labelled `agent` in Gmail were archived by the agent. Review them in Gmail by searching `label:agent`.

## Phase B (Future)

When the agent's decisions are trusted, this will be automated via a Python script + Anthropic SDK. The logic in `agent_prompt.md` will be translated 1:1 into the Python orchestrator.
