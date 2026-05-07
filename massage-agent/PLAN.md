# Massage Booking Agent — Plan

## What This Does
Dana periodically sends emails with "עיסויים" in the subject, linking to a Google Sheets spreadsheet of available massage time slots. Slots fill up fast. This agent monitors for those emails, finds the best available slot (calendar-aware, never before 10:30), books it by writing your name into the spreadsheet, creates a private calendar event, and confirms via Slack — all automatically within minutes of the email arriving.

---

## Architecture

Two new files + one small edit to the existing email triage:

```
massage-agent/
  agent_prompt.md    ← full booking agent logic (new)
  state.md           ← runtime state / processed email IDs (new)
email-agent/agent_prompt.md  ← Check 2b: also write a trigger when massage email detected (small edit)
```

**How the two agents connect:**
The email triage already detects massage emails (Check 2b → Slack DM). It will additionally write a "pending trigger" entry to `massage-agent/state.md`. A CronCreate scheduled job polls every 5 minutes, reads the trigger, and runs the booking flow. If no pending trigger exists, the agent exits in ~1 second — the poll is essentially free.

---

## Booking Agent Logic (Step by Step)

### Step 0 — Guard
Read `massage-agent/state.md`.
- `pending_message_id` empty → exit, nothing to do.
- `pending_message_id` already in `processed` list → clear pending, exit (already handled).

### Step 1 — Fetch Email Body
Fetch the full email from Gmail. Decode the body. Extract the Google Sheets URL
(`https://docs.google.com/spreadsheets/d/<id>`).

**Fail:** No URL found → Slack DM alert, mark processed, exit.

### Step 2 — Read Spreadsheet
Get sheet tab names, then download all data. Parse intelligently:
- Find the column with available time slots (by time/date patterns in headers)
- Find the "name" column (empty = available; headers like "שם", "name", "booked by")
- A slot is available if its name cell is empty

**Fail:** Spreadsheet unreadable or structure unrecognizable → Slack DM alert, mark processed, exit.

### Step 3 — Filter Slots
- Keep only slots where the name cell is empty
- Build ISO datetimes using Israel timezone (`Asia/Jerusalem`)
- **Drop any slot before 10:30**

**No slots remain** → Step 6 (no slot).

### Step 4 — Calendar Conflict Check
For each candidate slot, check Google Calendar:
- Window: `slot_start` to `slot_start + 30 minutes`
- A slot is conflicted if any non-cancelled calendar event overlaps that window

**Selection order:**
1. First, try post-lunch slots (≥ 13:00) — pick earliest available
2. If none available after 13:00, fall back to earliest available slot ≥ 10:30

**All slots conflicted** → Step 6 (no slot).

### Step 5 — Book
**5a — Write name to spreadsheet**
Write "Uri Danan" into the chosen slot's cell using the Python temp-file pattern
(required to avoid a zsh `!` escaping bug with Google Sheets range notation).

If the API write fails → Slack DM, do NOT mark processed (agent will retry on next poll). Exit.

**5b — Create private calendar event**
```
Summary: עיסוי
Start: <slot time>
End: <slot time + 30 min>
Visibility: private
Calendar: primary
```
If calendar create fails after the sheet write → Slack DM noting partial success, mark processed
(can't re-book since name is already written in the sheet).

### Step 6 — No Slot Found
```
😔 Massage booking — no available slot found.
[All slots before 10:30 or all slots conflict with your calendar.]
<link to spreadsheet>
```
Mark processed. You can book manually from the link.

### Step 7 — Success
```
💆 Massage booked!
Date: YYYY-MM-DD
Time: HH:MM
Slot: <human-readable slot description>
<link to spreadsheet>
```

### Step 8 — Update State
Append message ID to `processed`. Clear pending fields. Record booking details and outcome.

---

## State File Design (`massage-agent/state.md`)

```markdown
# Massage Agent State

## Processed Email IDs
- processed: []

## Pending Trigger
- pending_message_id: ""
- pending_gmail_url: ""
- pending_subject: ""
- triggered_at: ""

## Last Booking
- booked_slot: ""
- booked_date: ""
- calendar_event_id: ""
- spreadsheet_id: ""
- booking_confirmed_at: ""

## Last Run
- timestamp: ""
- outcome: ""    # booked | skipped_no_pending | skipped_already_processed | skipped_no_slot | failed_no_spreadsheet | failed_sheets | failed_write_sheet | partial_calendar_failed
- notes: ""
```

The `processed` list is the double-booking guard. Once a message ID is in that list, the agent will never act on it again regardless of how many poll cycles fire.

---

## Failure Handling Summary

| Scenario | Slack alert? | Mark processed? | Retry? |
|---|---|---|---|
| No spreadsheet URL in email | ✅ | ✅ | ❌ Manual |
| Spreadsheet unreadable | ✅ | ✅ | ❌ Manual |
| No slots ≥ 10:30 or all conflict | ✅ with sheet link | ✅ | ❌ Manual |
| Sheet write API error | ✅ | ❌ | ✅ Next poll |
| Calendar fails after sheet write | ✅ (partial) | ✅ | ❌ (sheet already written) |

---

## Confirmed Parameters
- **Session duration:** 30 minutes
- **Slot preference:** Post-lunch (≥ 13:00) first; fall back to earliest ≥ 10:30
- **No-slot fallback:** Alert once, mark processed, stop

---

## Scheduling
CronCreate job (`*/5 * * * *`) running the massage agent every 5 minutes.
No-op cost when idle: ~1 second (read state.md, check field, exit).

---

## Open Decisions for You to Make

### 1. Should the email triage trigger always write to massage-agent/state.md?
Today the triage only runs when you manually invoke `/email-triage`. If you run the triage
infrequently, the massage email might sit in your inbox for hours before the trigger is written.

**Options:**
- **A) Keep triage manual, add massage check to it** — Simple, but booking only happens when you run the triage. Fast enough if you run triage daily.
- **B) Set up triage on a cron too** — Both triage and massage agent run automatically. More infrastructure.
- **C) Massage agent independently checks Gmail** — The massage agent polls Gmail directly for Dana's emails (no dependency on the triage agent). Simpler to reason about, fully autonomous.

Option C is arguably cleanest: the massage agent searches Gmail itself
(`from:(danag@crazylabs.com OR danag@tabtale.com) subject:עיסויים is:unread`) on every poll.
No coupling to the triage agent, no shared trigger file needed.

### 2. Do you want a confirmation step before booking?
Currently the plan is **fully automatic** — it books without asking you first.
Alternative: the agent sends you a Slack message with the chosen slot and waits for a ✅ reaction
before writing to the spreadsheet. Safer but slower (defeats the "slots fill fast" goal).

### 3. Massage agent scope: Claude Code slash command or standalone Python script?
- **Slash command** (like the current email triage) — same pattern, no new infrastructure, runs inside Claude Code
- **Python + Anthropic SDK script** — runs independently, doesn't require Claude Code to be open

---

## Files to Create/Edit (Implementation)
| File | Action |
|---|---|
| `massage-agent/agent_prompt.md` | Create — full booking logic |
| `massage-agent/state.md` | Create — initial empty state |
| `email-agent/agent_prompt.md` | Edit — Check 2b trigger write (if Option A/B chosen) |

---

*Come back with your answers to the 3 Open Decisions above and we can start building.*
