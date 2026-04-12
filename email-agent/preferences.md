# Email Agent Preferences

## Identity
- **Primary email:** urid@crazylabs.com
- **Alias:** urid@tabtale.com
- **Name:** Uri D.

## VIP Senders
Emails from these people trigger an immediate Slack DM alert:
- Sagi Shliesser — sagis@tabtale.com / sagis@crazylabs.com
- Guy T. — guyt@tabtale.com / guyt@crazylabs.com
- Ariel V. — arielv@tabtale.com / arielv@crazylabs.com

## Urgency Keywords
Flag as urgent if subject or body contains:
urgent, ASAP, time-sensitive, deadline, EOD, "by tomorrow", "action required"

## Archive Rules
- **Newsletters:** Archive, EXCEPT educational ones (e.g. thebatch@deeplearning.ai)
- **System notifications:** Archive UNLESS there is an action item addressed to me
- **CC'd emails:** Archive if I am not mentioned by name in the body → add to digest
- **JIRA/GitHub:** Archive UNLESS I am mentioned or ticket is assigned to me → add to digest with link
- **Meeting accepts:** Always archive
- **Meeting declines:** Archive UNLESS I organized the meeting
- **Social media notifications:** Archive
- **Company-wide announcements:** Do NOT archive
- **Labeling:** Apply the Gmail label `agent` to every archived email for auditability

## Email Answering
- Mode: **draft-only** (no auto-send until further notice)
- Answer sources: email chain, Google Drive

## Slack Output
- **Alerts:** DM to myself — VIP and urgent emails
- **Canvas:** Same DM, updated in place each run
  - Sections: Action Items (checklist) · CC'd Archived · JIRA/GitHub · Archived Summary
