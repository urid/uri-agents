# Email Agent — Design Spec
**Date:** 2026-04-12
**Author:** Uri D.
**Status:** Approved

---

## Overview

An AI agent that triages the inbox of urid@crazylabs.com (alias: urid@tabtale.com), reducing manual email processing to only what truly requires human attention. The agent archives noise, drafts replies, extracts action items, and alerts on high-priority emails.

---

## Architecture

### Phase A: Interactive (current)
Run manually as a Claude Code session. Claude uses MCP tools directly to read Gmail, act on emails, and update Slack. No additional infrastructure required.

### Phase B: Autonomous (future)
A Python script using the Anthropic SDK and scheduled via cron/Task Scheduler. The same logic runs unattended on a defined interval. Phase A validates behavior before automating.

### Components

```
Inbox Fetcher        → google_workspace_mcp (Gmail API)
Triage Engine        → Claude classifies each unread email
Gmail Actions        → archive, label, create draft (google_workspace_mcp)
Drive Lookup         → search for answer context (google_workspace_mcp)
Slack Updater        → send alerts, update canvas (Slack MCP)
Preferences          → email-agent/preferences.md (read at session start)
```

### MCP Servers
- **google_workspace_mcp** (taylorwilsdon) — Gmail + Drive access via OAuth2
- **Slack MCP** (already configured) — DM alerts + canvas updates

---

## Triage Decision Rules

Rules are evaluated in order. Multiple outcomes can apply to a single email.

### 1. VIP Sender
If sender matches any VIP email → send Slack DM alert immediately. Continue evaluating (do not auto-archive).

### 2. Urgent
If subject or body contains urgency keywords → send Slack DM alert immediately. Continue evaluating.

Urgency keywords: `urgent`, `ASAP`, `time-sensitive`, `deadline`, `EOD`, `by tomorrow`, `action required`

### 3. Archive Check (first match wins)
| Condition | Action |
|-----------|--------|
| Marketing/newsletter (except educational) | Archive + label `agent` |
| System notification with no action item for me | Archive + label `agent` |
| CC'd and I am not mentioned by name in body | Archive + label `agent` + digest (CC section) |
| JIRA/GitHub and I am not mentioned and ticket not assigned to me | Archive + label `agent` + digest (JIRA section with link) |
| Meeting accept reply | Archive + label `agent` |
| Meeting decline reply (not for meetings I organized) | Archive + label `agent` |
| Social media notification | Archive + label `agent` |
| Company-wide announcement | **Do not archive** |

All archived emails receive the Gmail label `agent`. The label is created on first run if it does not exist.

### 4. Action Item
If the email directly addresses me with a task, request, or decision needed → add to canvas Action Items checklist.

### 5. Answerable
If a clear answer exists in the email chain or Google Drive → create Gmail draft reply → add to canvas Action Items (flagged as "draft ready").

All other emails remain in inbox, untouched.

---

## Slack Output

All output is delivered to a **DM to self** in Slack.

### Real-time Alerts (DM messages)
Sent immediately for VIP and urgent emails:
```
🔴 VIP Email — Sagi Shliesser
   From: sagis@tabtale.com
   Subject: Q2 Budget Review
   Preview: "Uri, I need your sign-off on..."
   → [Open in Gmail]
```

### Canvas (persistent, updated in place each run)
A single canvas in the DM, never recreated — updated on every agent run.

Sections:
1. **Action Items** — checklist; items persist until manually cleared
2. **CC'd (Archived)** — replaced each run; summary of CC'd emails that were archived because I was not directly mentioned in the body
3. **JIRA / GitHub** — replaced each run; ticket links where I am assigned or mentioned
4. **Archived Summary** — replaced each run; count and category breakdown of what was archived

---

## Email Answering

- **Mode:** Draft-only (no auto-send)
- Answer sources: email chain history, Google Drive
- Draft is saved to Gmail Drafts; flagged in canvas Action Items as "draft ready"
- Mode can be upgraded to `auto_send` in preferences.md once behavior is validated

---

## Preferences File

Stored at `email-agent/preferences.md`. Read by the agent at the start of every run. Human-editable — all behavior changes are made here, not in code.

See: [email-agent/preferences.md](../../email-agent/preferences.md)

---

## VIP Senders

| Name | Emails |
|------|--------|
| Sagi Shliesser | sagis@tabtale.com, sagis@crazylabs.com |
| Guy T. | guyt@tabtale.com, guyt@crazylabs.com |
| Ariel V. | arielv@tabtale.com, arielv@crazylabs.com |

---

## Out of Scope (Phase A)

- Sending emails (draft-only)
- Calendar management
- Multi-account support beyond urid@crazylabs.com / urid@tabtale.com
- Slack channels other than DM to self
- Learning/feedback loop (future phase)
