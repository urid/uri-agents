# Email Triage Agent — Phase A Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Set up an interactive Claude Code session that triages urid@crazylabs.com's inbox using Gmail + Drive MCP tools, archives noise, creates Gmail drafts, and outputs alerts/digest to a persistent Slack canvas.

**Architecture:** Claude Code reads `email-agent/preferences.md` and `email-agent/state.md`, fetches unread Gmail via google_workspace_mcp, applies triage rules per the agent prompt, executes Gmail actions (archive, label, draft), and updates a persistent Slack canvas via the existing Slack MCP.

**Tech Stack:** google_workspace_mcp (Python/uv, taylorwilsdon/google_workspace_mcp), Slack MCP (already configured), Claude Code interactive session (Phase A)

---

## File Map

| File | Status | Purpose |
|------|--------|---------|
| `email-agent/preferences.md` | exists | Agent config — VIP list, archive rules, all behavior |
| `email-agent/state.md` | create | Runtime state — Slack DM channel ID, canvas ID, last run info |
| `email-agent/agent_prompt.md` | create | Full triage instructions Claude reads each session |
| `email-agent/README.md` | create | How to run the agent |
| `~/.claude/settings.json` | modify | Add google_workspace_mcp MCP server config |

---

## Task 1: Install uv on Windows

**Files:** none (system-level install)

- [ ] **Step 1: Check if uv is already installed**

Run in terminal:
```bash
uv --version
```
Expected: version string like `uv 0.5.x`. If you see this, skip to Task 2.

- [ ] **Step 2: Install uv (if not present)**

Run in PowerShell (as normal user, not admin):
```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```
Expected output: `Downloading uv ... Installed uv to C:\Users\urida\.cargo\bin\uv`

- [ ] **Step 3: Verify install**

Close and reopen terminal, then:
```bash
uv --version
```
Expected: `uv 0.5.x` (or newer)

- [ ] **Step 4: Commit (nothing to commit — system install only)**

No files changed. Proceed to Task 2.

---

## Task 2: Clone and install google_workspace_mcp

**Files:** none in repo (cloned to `C:\Users\urida\tools\google_workspace_mcp`)

- [ ] **Step 1: Create tools directory and clone**

```bash
mkdir -p /c/Users/urida/tools
cd /c/Users/urida/tools
git clone https://github.com/taylorwilsdon/google_workspace_mcp.git
```
Expected: repo cloned to `C:\Users\urida\tools\google_workspace_mcp`

- [ ] **Step 2: Install Python dependencies**

```bash
cd /c/Users/urida/tools/google_workspace_mcp
uv sync
```
Expected: `Resolved N packages` and `Installed N packages`

- [ ] **Step 3: Verify it starts**

```bash
uv run main.py --help
```
Expected: help text showing `--single-user`, `--tools`, `--transport` flags. No errors.

---

## Task 3: Create Google Cloud project and OAuth credentials

**Files:** `C:\Users\urida\tools\google_workspace_mcp\.env` (created here, NOT in the repo)

This task is done in a browser and text editor. Follow these steps exactly.

- [ ] **Step 1: Create a Google Cloud project**

1. Open https://console.cloud.google.com/
2. Click the project dropdown at the top → **New Project**
3. Name it `uri-email-agent` → **Create**
4. Wait for creation, then select the new project from the dropdown

- [ ] **Step 2: Enable Gmail API**

1. In the Cloud Console, go to **APIs & Services → Library**
2. Search for `Gmail API` → click it → **Enable**
3. Wait for "API enabled" confirmation

- [ ] **Step 3: Enable Google Drive API**

1. Back in **APIs & Services → Library**
2. Search for `Google Drive API` → click it → **Enable**

- [ ] **Step 4: Create OAuth2 credentials**

1. Go to **APIs & Services → Credentials**
2. Click **+ Create Credentials → OAuth client ID**
3. If prompted to configure consent screen: click **Configure consent screen**
   - User type: **Internal** (your org) if available, else External
   - App name: `Uri Email Agent`
   - User support email: `urid@crazylabs.com`
   - Click **Save and Continue** through remaining screens
4. Back at Create OAuth client ID:
   - Application type: **Desktop app**
   - Name: `uri-email-agent-desktop`
   - Click **Create**
5. A dialog shows your **Client ID** and **Client Secret** — copy both

- [ ] **Step 5: Create .env file with credentials**

Create `C:\Users\urida\tools\google_workspace_mcp\.env` with this content (replace with your actual values):

```
GOOGLE_OAUTH_CLIENT_ID=YOUR_CLIENT_ID_HERE.apps.googleusercontent.com
GOOGLE_OAUTH_CLIENT_SECRET=YOUR_CLIENT_SECRET_HERE
```

Do NOT commit this file. It lives only in the tools directory.

---

## Task 4: Configure Claude Code to use google_workspace_mcp

**Files:** `C:\Users\urida\.claude\settings.json` (modify)

- [ ] **Step 1: Read current settings.json**

Read `C:\Users\urida\.claude\settings.json` to see current state before editing.

- [ ] **Step 2: Add mcpServers block**

Edit `C:\Users\urida\.claude\settings.json` to add the `mcpServers` key. The final file should look like:

```json
{
  "permissions": {
    "defaultMode": "auto"
  },
  "enabledPlugins": {
    "superpowers@claude-plugins-official": true,
    "frontend-design@claude-plugins-official": true,
    "gws@talb-plugins": true
  },
  "extraKnownMarketplaces": {
    "superpowers-marketplace": {
      "source": {
        "source": "github",
        "repo": "obra/superpowers-marketplace"
      }
    },
    "talb-plugins": {
      "source": {
        "source": "github",
        "repo": "TabTale/claude-plugins"
      }
    }
  },
  "autoUpdatesChannel": "latest",
  "skipAutoPermissionPrompt": true,
  "mcpServers": {
    "google-workspace": {
      "command": "uv",
      "args": [
        "run",
        "main.py",
        "--single-user",
        "--tools",
        "gmail",
        "drive"
      ],
      "cwd": "C:\\Users\\urida\\tools\\google_workspace_mcp",
      "env": {
        "GOOGLE_OAUTH_CLIENT_ID": "YOUR_CLIENT_ID_HERE.apps.googleusercontent.com",
        "GOOGLE_OAUTH_CLIENT_SECRET": "YOUR_CLIENT_SECRET_HERE"
      }
    }
  }
}
```

Replace `YOUR_CLIENT_ID_HERE` and `YOUR_CLIENT_SECRET_HERE` with the actual values from Task 3.

- [ ] **Step 3: Restart Claude Code**

Close and reopen Claude Code (or run `/mcp` to reload). The google-workspace MCP server should appear.

- [ ] **Step 4: Complete OAuth flow on first launch**

When Claude Code starts the MCP server for the first time, it will print a URL in its logs. Open that URL in your browser and approve access for:
- Gmail (read, modify, compose)
- Google Drive (read)

The token is stored automatically in `~/.google_workspace_mcp/` and auto-refreshes.

- [ ] **Step 5: Verify tools are available**

In a new Claude Code session, ask:
```
What Google Workspace tools do you have available?
```
Expected: Claude lists Gmail tools (search, get message, archive, create label, create draft) and Drive tools (search files, get file content).

Note the exact tool names — you'll use them in agent_prompt.md. Common names are `search_gmail_messages`, `get_gmail_message`, `modify_gmail_message`, `create_gmail_label`, `create_gmail_draft`, `search_drive_files`.

- [ ] **Step 6: Verify email access**

Ask Claude Code:
```
Search my Gmail for the 3 most recent unread emails using the Google Workspace MCP. Just show sender and subject.
```
Expected: Claude returns 3 real emails from your inbox.

---

## Task 5: Find your Slack user ID and DM channel

**Files:** `email-agent/state.md` (create)

- [ ] **Step 1: Find your Slack user ID**

In a Claude Code session with Slack MCP:
```
Use the Slack MCP to search for the user with email urid@crazylabs.com or name "Uri". Return the user ID.
```
Expected: A user ID like `U01ABCDE123`. Note it down.

- [ ] **Step 2: Open / confirm DM to self**

```
Send a Slack DM to myself (user ID [YOUR_ID]) saying: "Email agent initializing..."
```
Expected: Message sent. Claude returns the DM channel ID — it starts with `D` (e.g., `D01ABCDE456`). Note it down.

- [ ] **Step 3: Create state.md**

Create `email-agent/state.md` with the actual values:

```markdown
# Email Agent State

## Slack
- **DM channel ID:** D01ABCDE456
- **Canvas ID:** (set in Task 6)

## Last Run
- **Timestamp:** never
- **Emails processed:** 0
- **Emails archived:** 0
- **Alerts sent:** 0
- **Drafts created:** 0
```

Replace `D01ABCDE456` with your actual DM channel ID.

---

## Task 6: Create the Slack canvas

**Files:** `email-agent/state.md` (update with canvas ID)

- [ ] **Step 1: Create the canvas in your DM**

In Claude Code:
```
Using the Slack MCP, create a canvas in my DM channel [YOUR_DM_CHANNEL_ID] with this initial content:

# Email Digest — Initializing

## ✅ Action Items
(no items yet)

## 📧 CC'd (Archived)
(none this run)

## 🎫 JIRA / GitHub
(none this run)

## 🗄️ Archived This Run
(no runs yet)
```
Expected: Canvas created. Claude returns a canvas ID like `F01ABCDE789`. Note it down.

- [ ] **Step 2: Update state.md with canvas ID**

Edit `email-agent/state.md` — replace `(set in Task 6)` with the actual canvas ID:

```markdown
# Email Agent State

## Slack
- **DM channel ID:** D01ABCDE456
- **Canvas ID:** F01ABCDE789

## Last Run
- **Timestamp:** never
- **Emails processed:** 0
- **Emails archived:** 0
- **Alerts sent:** 0
- **Drafts created:** 0
```

- [ ] **Step 3: Verify canvas is visible**

Open Slack, go to your DM with yourself, confirm the canvas appears with the initial content.

- [ ] **Step 4: Commit state.md**

```bash
cd "D:/Github/uri-agents"
git add email-agent/state.md
git commit -m "feat: initialize Slack canvas and state for email agent"
```

---

## Task 7: Write agent_prompt.md

**Files:**
- Create: `email-agent/agent_prompt.md`

This is the core Phase A artifact — the complete instructions Claude reads at the start of every triage session.

- [ ] **Step 1: Create email-agent/agent_prompt.md with this exact content**

```markdown
# Email Triage Agent

You are Uri D.'s email triage assistant for urid@crazylabs.com (also urid@tabtale.com).

## Startup

1. Read `email-agent/preferences.md` — this is your configuration.
2. Read `email-agent/state.md` — this has your Slack canvas ID and DM channel.

---

## Step 1: Fetch Unread Emails

Search Gmail for: `is:unread in:inbox`

Fetch up to 50 messages. For each, collect:
- `message_id` — needed for all Gmail actions
- `thread_id` — for Gmail URL and draft replies
- `from_email` — sender's email (lowercase)
- `from_name` — sender's display name
- `to_list` — To recipients (is urid@crazylabs.com or urid@tabtale.com in To?)
- `cc_list` — CC recipients (is urid@crazylabs.com or urid@tabtale.com in CC?)
- `subject`
- `body_snippet` — first 500 characters of body text
- `date`

Construct Gmail URL for each email: `https://mail.google.com/mail/u/0/#inbox/[thread_id]`

---

## Step 2: Triage Each Email

Initialize these lists (start empty each run):
- `vip_alerts` — emails needing instant Slack DM alert (VIP sender)
- `urgent_alerts` — emails needing instant Slack DM alert (urgency keywords, non-VIP only)
- `to_archive` — list of {message_id, reason}
- `cc_digest` — list of {sender_display, subject, summary}
- `jira_digest` — list of {ticket_id, link, description}
- `action_items` — list of {description, draft_id (optional)}

For EACH email, run ALL checks in order. Multiple outcomes can apply.

---

### Check 1 — VIP Sender

Does `from_email` (case-insensitive) match:
- sagis@tabtale.com OR sagis@crazylabs.com
- guyt@tabtale.com OR guyt@crazylabs.com
- arielv@tabtale.com OR arielv@crazylabs.com

If YES → append to `vip_alerts`:
```
{from_name, from_email, subject, preview: body_snippet[:200], gmail_url}
```
Continue to Check 2 — do NOT skip remaining checks.

---

### Check 2 — Urgency Keywords

Does `subject` OR `body_snippet` contain (case-insensitive):
`urgent`, `asap`, `time-sensitive`, `deadline`, `eod`, `by tomorrow`, `action required`

If YES → append to `urgent_alerts`:
```
{from_name, from_email, subject, preview: body_snippet[:200], gmail_url}
```
Continue to Check 3 — do NOT skip remaining checks.

---

### Check 3 — Archive Rules (first match wins, then skip to Check 4)

**3a — Newsletter / Marketing**
Archive if NOT an educational sender AND at least one marketing signal:

Educational exceptions (never archive): thebatch@deeplearning.ai, any @deeplearning.ai, @substack.com newsletters that appear technical/educational (use judgment on topic).

Marketing signals (archive if any present):
- Subject contains: "% off", "sale ends", "deal", "promo", "unsubscribe", "newsletter", "digest", "weekly roundup", "monthly update"
- Body contains the word "unsubscribe"
- `from_email` domain is one of: mailchimp.com, sendgrid.net, campaign-monitor.com, hubspot.com, constantcontact.com, klaviyo.com, mailerlite.com, convertkit.com, beehiiv.com

→ `to_archive`: {message_id, reason: "newsletter/marketing"}. **Skip 3b–3g.**

---

**3b — System Notification (no action item for me)**
Archive if:
- Sender looks automated: `from_email` starts with or equals: noreply, no-reply, automated, notifications, alerts, mailer-daemon, postmaster, do-not-reply, support (from ticketing systems), info@ (from platforms)
- AND body does NOT contain "Uri" or "urid" followed by a task/request within 50 characters

→ `to_archive`: {message_id, reason: "system notification"}. **Skip 3c–3g.**

---

**3c — CC'd, not mentioned by name**
Archive if:
- I am in `cc_list` (not `to_list`)
- `body_snippet` does NOT contain "Uri" or "urid" (case-insensitive)

→ `to_archive`: {message_id, reason: "CC not mentioned"}
→ `cc_digest`: {sender_display: from_name + " <" + from_email + ">", subject, summary: one sentence describing what the email is about}
**Skip 3d–3g.**

---

**3d — JIRA / GitHub auto-notification**
Archive if:
- `from_email` matches: *@atlassian.net, jira@*, noreply@github.com, notifications@github.com
- OR subject matches JIRA pattern: contains `[PROJ-` or `[` followed by letters, dash, digits, `]`

AND NOT:
- Body contains "Assignee:" or "assigned to" near "Uri" or "urid"
- Body contains "@urid"
- Body contains "mentioned you" (GitHub mention)

→ `to_archive`: {message_id, reason: "JIRA/GitHub auto"}
→ Extract ticket ID (regex `\[([A-Z]+-\d+)\]` from subject) and link (first https://... in body matching jira or github)
→ `jira_digest`: {ticket_id, link, description: subject with ticket prefix stripped}
**Skip 3e–3g.**

---

**3e — Meeting accept reply**
Archive if subject (case-insensitive) starts with "Accepted:" or contains "has accepted your invitation":

→ `to_archive`: {message_id, reason: "meeting accept"}. **Skip 3f–3g.**

---

**3f — Meeting decline (not my meeting)**
Archive if:
- Subject starts with "Declined:" or contains "has declined your invitation" or "won't be attending"
- AND I am NOT the meeting organizer (check body for "Organizer:" — if organizer is urid@crazylabs.com or urid@tabtale.com, do NOT archive)

→ `to_archive`: {message_id, reason: "meeting decline"}. **Skip 3g.**

---

**3g — Social media**
Archive if `from_email` domain is: linkedin.com, twitter.com, x.com, facebook.com, instagram.com, youtube.com, tiktok.com, pinterest.com

→ `to_archive`: {message_id, reason: "social media"}

---

**If none of 3a–3g matched:** Email stays in inbox. Proceed to Check 4.

---

### Check 4 — Action Item

Does this email directly ask me (Uri) to do something, decide something, or respond?

Signals: "Uri, can you", "please review", "could you", "your approval needed", "waiting on you", "need your input", "can you confirm", direct question to me in subject or body, "please advise", deadline on a task for me.

If YES → append to `action_items`:
```
{description: "Reply to [from_name] re: [subject] — [one line: what's needed]"}
```

---

### Check 5 — Answerable

Can a complete, correct reply be written from:
a) The email chain history (fetch full thread if needed), OR
b) A Google Drive document (search Drive for relevant content)

Only proceed if the answer is CLEAR and CERTAIN. If there is any ambiguity, skip.

If YES:
1. Fetch full thread if needed
2. Search Drive if needed: use subject/topic as query
3. Create Gmail draft reply with the answer
4. Append to `action_items`:
```
{description: "[DRAFT READY] Reply to [from_name] re: [subject]", draft_id: "[id]"}
```

---

## Step 3: Execute Gmail Actions

For all emails in `to_archive`:

1. Check if label "agent" exists. If not, create it.
2. For each email: add label "agent" AND remove from INBOX (archive).

Process in batches of 10 to avoid rate limits. If a batch fails, log the failure and continue with the next batch.

---

## Step 4: Send Slack Alerts

Use Slack DM channel ID from state.md.

For each email in `vip_alerts`, send this message:
```
🔴 *VIP Email — [from_name]*
From: [from_email]
Subject: [subject]
> [preview, max 150 chars]
<[gmail_url]|Open in Gmail>
```

For each email in `urgent_alerts` that is NOT already in `vip_alerts`:
```
⚡ *Urgent Email — [from_name]*
From: [from_email]
Subject: [subject]
> [preview, max 150 chars]
<[gmail_url]|Open in Gmail>
```

---

## Step 5: Update Slack Canvas

1. Read current canvas using canvas_id from state.md.
2. Find and extract everything between `## ✅ Action Items` and the next `##` header — this is the preserved action items block.
3. Build updated canvas:

```
# Email Digest — Last updated: [YYYY-MM-DD HH:MM]

## ✅ Action Items
[PRESERVED EXISTING ITEMS — copy exactly from current canvas]
[NEW ITEMS from action_items list — one line each:]
- [ ] [description]

## 📧 CC'd (Archived)
[For each entry in cc_digest:]
- **[sender_display]** — *[subject]*: [summary]
[If cc_digest is empty: write "(none this run)"]

## 🎫 JIRA / GitHub
[For each entry in jira_digest:]
- [[ticket_id]]([link]) — [description]
[If jira_digest is empty: write "(none this run)"]

## 🗄️ Archived This Run ([total count of to_archive] emails)
- Newsletters/marketing: [count]
- System notifications: [count]
- CC'd (not mentioned): [count]
- JIRA/GitHub auto: [count]
- Meeting accepts: [count]
- Meeting declines: [count]
- Social media: [count]
```

4. Call `slack_update_canvas` with canvas_id and the full new content.

---

## Step 6: Update state.md

Rewrite `email-agent/state.md`:

```markdown
# Email Agent State

## Slack
- **DM channel ID:** [unchanged]
- **Canvas ID:** [unchanged]

## Last Run
- **Timestamp:** [current ISO timestamp, e.g. 2026-04-12T09:00:00]
- **Emails processed:** [count]
- **Emails archived:** [count of to_archive]
- **Alerts sent:** [count of vip_alerts + urgent_alerts]
- **Drafts created:** [count of drafts]
```

---

## Final Report

Output a brief summary:
```
Run complete.
- Processed: N emails
- Archived: N (newsletters: N, system: N, CC: N, JIRA: N, meeting accepts: N, meeting declines: N, social: N)
- VIP alerts: N
- Urgent alerts: N
- Action items added: N
- Drafts created: N
```
```

- [ ] **Step 2: Commit agent_prompt.md**

```bash
cd "D:/Github/uri-agents"
git add email-agent/agent_prompt.md
git commit -m "feat: add email triage agent prompt (Phase A)"
```

---

## Task 8: Write README.md

**Files:**
- Create: `email-agent/README.md`

- [ ] **Step 1: Create email-agent/README.md**

```markdown
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
```

- [ ] **Step 2: Commit README.md**

```bash
cd "D:/Github/uri-agents"
git add email-agent/README.md
git commit -m "feat: add email agent README with run instructions"
```

---

## Task 9: Dry-run — observe triage decisions without acting

**Files:** none changed

This task validates that triage decisions look correct before taking any real actions.

- [ ] **Step 1: Start a dry-run session**

In Claude Code:
```
Read email-agent/agent_prompt.md. Run Steps 1 and 2 only (fetch emails and triage).
Do NOT execute any Gmail actions, do NOT send Slack messages, do NOT update the canvas.
Instead, show me a table of the first 20 unread emails with columns:
- Subject
- From
- Triage decision (what would happen: archive reason / action item / answerable / leave in inbox)
```

- [ ] **Step 2: Review the decisions**

Read the table. For each entry, verify:
- Newsletters you don't care about → archive ✓
- Emails from VIPs → alert ✓
- CC'd emails where you're not mentioned → archive + CC digest ✓
- Emails requiring your response → action item ✓
- Educational newsletters (thebatch@deeplearning.ai etc.) → leave in inbox ✓

- [ ] **Step 3: Fix any misclassifications**

If the agent made wrong decisions, update `email-agent/preferences.md` with the correction and note why. Example: if a specific system notification sender should be excluded, add it to the educational exceptions list or add a note in preferences.md.

- [ ] **Step 4: Commit any preferences updates**

```bash
cd "D:/Github/uri-agents"
git add email-agent/preferences.md
git commit -m "fix: adjust archive rules based on dry-run observations"
```

(Only run this step if changes were made.)

---

## Task 10: First full run

**Files:** `email-agent/state.md` (updated by agent)

- [ ] **Step 1: Run the full agent**

In Claude Code:
```
Read email-agent/agent_prompt.md and run the full email triage agent — all steps including archiving, Slack alerts, and canvas update.
```

- [ ] **Step 2: Verify Gmail label created**

In Gmail, search: `label:agent`
Expected: archived emails appear, all with the `agent` label.

- [ ] **Step 3: Verify canvas updated**

Open Slack, go to your DM with yourself, open the canvas.
Expected: digest sections updated with this run's data. Action items present.

- [ ] **Step 4: Verify alerts sent**

Check your Slack DM messages (not canvas).
Expected: if any VIP or urgent emails were in inbox, alerts appear as DM messages.

- [ ] **Step 5: Verify state.md updated**

Read `email-agent/state.md`. Expected: last run timestamp is current, counts are non-zero.

- [ ] **Step 6: Commit state.md**

```bash
cd "D:/Github/uri-agents"
git add email-agent/state.md
git commit -m "chore: update state after first email agent run"
```

---

## Self-Review Notes

**Spec coverage check:**
- ✅ Archive with criteria (rules 3a–3g) → Task 7 (agent_prompt.md Step 2 Check 3)
- ✅ Gmail `agent` label applied to all archived emails → Task 7 Step 3
- ✅ Answer emails as draft-only → Task 7 Step 2 Check 5
- ✅ Extract action items → Task 7 Step 2 Check 4
- ✅ VIP alerts (sagis, guyt, arielv @tabtale + @crazylabs) → Task 7 Step 2 Check 1
- ✅ Urgent keyword alerts → Task 7 Step 2 Check 2
- ✅ Slack DM alerts → Task 7 Step 4
- ✅ Slack canvas (persistent, updated in place) → Task 7 Step 5
- ✅ CC Archived section in canvas → Task 7 Step 5 (cc_digest)
- ✅ JIRA section with links in canvas → Task 7 Step 5 (jira_digest)
- ✅ Action Items checklist preserved across runs → Task 7 Step 5 (preserved block)
- ✅ Educational newsletters (thebatch@deeplearning.ai) NOT archived → Task 7 Step 2 Check 3a
- ✅ Meeting accepts always archived → Task 7 Step 2 Check 3e
- ✅ Meeting declines archived unless organizer is me → Task 7 Step 2 Check 3f
- ✅ google_workspace_mcp setup → Tasks 1–4
- ✅ Slack canvas initialized → Tasks 5–6
- ✅ state.md tracks canvas ID and DM channel → Task 5 + 6
- ✅ Dry-run before full run → Task 9
