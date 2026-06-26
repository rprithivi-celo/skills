---
name: cs-daily-briefing
description: Set up 3 automated daily Slack briefings (8 AM, 12 PM, 4 PM) for Celonis CS engineers — scans support channels, enriches active incidents with CeloAssist context, and DMs a structured summary with source links.
this is a test skill for testing the CS Daily Briefing setup
---   
   
# CS Daily Briefing — Setup Skill    - testing

This skill sets up **three automated daily Slack briefings** tailored for Celonis Customer Support engineers. Each briefing scans key support and incident channels, enriches active incidents with CeloAssist historical context, and DMs you a structured summary with one-click source links.

**What you get:**
- ☀️ **8:00 AM** — Overnight catch-up: what happened while you were away
- 🕛 **12:00 PM** — Midday check-in: what's new since morning
- 🌆 **4:00 PM** — EOD wrap-up: what still needs action before you log off + watch list for tomorrow

**Prerequisites:**
- Slack plugin must be connected in Cowork
- Set Cowork to launch at login: `System Settings → General → Login Items`

---

## Step 1 — Detect your Slack ID

Call `mcp__slack__slack_read_user_profile` with no arguments to get the current user's profile. Extract the `user_id` field (format: `U...`). This is YOUR_SLACK_ID — substitute it into every task prompt below.

Tell the user: "Found your Slack ID: [YOUR_SLACK_ID] — setting up your 3 daily briefings now..."

---

## Step 2 — Create the three scheduled tasks

Use `mcp__scheduled-tasks__create_scheduled_task` for each task. Substitute YOUR_SLACK_ID where indicated. Create all three tasks **in parallel**. If a task with the same `taskId` already exists, use `mcp__scheduled-tasks__update_scheduled_task` instead.

---

### Task 1 — Morning Briefing (8 AM)

- **taskId:** `daily-slack-cs-briefing`
- **cronExpression:** `0 8 * * *`
- **description:** `Daily 8 AM Slack briefing: summarises overnight activity, enriches incidents with CeloAssist context, DMs the user`
- **prompt:**

```
You are running a daily Slack briefing for a Celonis Customer Support engineer. Their Slack user ID is YOUR_SLACK_ID. Scan the past ~24 hours, enrich active incidents with CeloAssist, and DM a concise briefing.

## Step 1 — Gather recent messages (run all in parallel)

Use mcp__slack__slack_search_public_and_private with after: set to yesterday's date (YYYY-MM-DD). For each search use limit=10 and response_format="detailed".

1. query: "incident SEV1 SEV2 after:YESTERDAY"
2. query: "escalation customer after:YESTERDAY in:#1st_level_customer_support"
3. query: "L2 queue after:YESTERDAY in:#2nd_level_customer_support"
4. query: "after:YESTERDAY in:#pne-customer-support-community"
5. query: "after:YESTERDAY in:#celostar-support"
6. query: "after:YESTERDAY in:#cs_tools"
7. query: "AI agent automation support after:YESTERDAY"

## Step 1.5 — Enrich with CeloAssist

Identify any active SEV1/SEV2 cases, platform issues, or service degradations from Step 1 results (case numbers, CBE IDs, error patterns, service names).

If any are found, send a DM to CeloAssist (Slack user ID: U08M89GMBST) using mcp__slack__slack_send_message. Ask: are there known issues, historical CBE cases, prior incidents, or workarounds matching these symptoms? List case IDs, symptoms, and services concisely.

Then wait 45 seconds: mcp__workspace__bash command: sleep 45

Then use mcp__slack__slack_read_thread with the channel_id and message_ts from the DM to read CeloAssist's response. If still "Thinking...", run sleep 20 and retry up to 2 times.

Skip this step if no active incidents found.

## Step 2 — Synthesise

Sections (skip empty ones):
- 🚨 Active Incidents / SEV1-2: major escalations. If CeloAssist found context, add a "CeloAssist context:" sub-line with key insight and CBE links.
- 📥 L2 Queue Updates: handoffs, OOO coverage needs
- ⚙️ Process & Tooling: tool issues, workflow changes
- 🤖 AI / Automation: AI agents, automation proposals
- 📣 Community Highlights: #pne-customer-support-community discussions

Keep bullets to 1-2 sentences. Include Jira/ticket links where mentioned. Flag items needing attention with 👀.
For every bullet, append → <permalink|Source> using the Permalink from the search result.

## Step 3 — Send DM

Use mcp__slack__slack_send_message to DM YOUR_SLACK_ID.
Start: "*☀️ Good morning — here's your daily CS briefing for [DATE]*"
End: "_This is your automated daily CS briefing. Reply to this message with any follow-ups._"
If nothing notable, still send the DM saying so — never skip.
```

---

### Task 2 — Midday Briefing (12 PM)

- **taskId:** `midday-slack-cs-briefing`
- **cronExpression:** `0 12 * * *`
- **description:** `Daily 12 PM midday Slack briefing: summarises morning activity, enriches new incidents with CeloAssist, DMs the user`
- **prompt:**

```
You are running a midday Slack briefing for a Celonis Customer Support engineer. Their Slack user ID is YOUR_SLACK_ID. Scan the past ~4 hours and DM a concise midday catchup.

## Step 1 — Gather recent messages (run in parallel)

Use mcp__slack__slack_search_public_and_private with after: set to today's date (YYYY-MM-DD). For each search use limit=8 and response_format="detailed".

1. query: "incident SEV1 SEV2 after:TODAY"
2. query: "after:TODAY in:#1st_level_customer_support"
3. query: "after:TODAY in:#2nd_level_customer_support"
4. query: "after:TODAY in:#pne-customer-support-community"
5. query: "after:TODAY in:#celostar-support"
6. query: "after:TODAY in:#cs_tools"

## Step 1.5 — Enrich with CeloAssist

Identify any NEW or ESCALATING incidents from Step 1. If found, DM CeloAssist (U08M89GMBST) asking about known issues, workarounds, or prior cases. Run sleep 45 via mcp__workspace__bash, then read the thread with mcp__slack__slack_read_thread. Retry up to 2 times if still thinking. Skip if no new incidents.

## Step 2 — Synthesise

Focus on what is NEW or CHANGED since morning:
- 🚨 New / Escalating Incidents: include "CeloAssist context:" sub-line if relevant
- 📥 L2 Activity: new tickets, handoffs
- ⚙️ Process / Tools: updates or issues flagged mid-morning
- 💬 Notable Discussions: active debates or decisions needed

Bullets 1-2 sentences. Flag with 👀.
For every bullet, append → <permalink|Source> using the Permalink from the search result.

## Step 3 — Send DM

Use mcp__slack__slack_send_message to DM YOUR_SLACK_ID.
Start: "*🕛 Midday check-in — [DATE]*"
End: "_Next briefing at 4 PM. Reply here with any follow-ups._"
```

---

### Task 3 — End of Day Briefing (4 PM)

- **taskId:** `eod-slack-cs-briefing`
- **cronExpression:** `0 16 * * *`
- **description:** `Daily 4 PM EOD Slack briefing: full day summary, flags action items before log-off, enriches open incidents with CeloAssist`
- **prompt:**

```
You are running an end-of-day Slack briefing for a Celonis Customer Support engineer. Their Slack user ID is YOUR_SLACK_ID. Scan the full day's activity and identify anything needing action before log-off.

## Step 1 — Gather today's messages (run in parallel)

Use mcp__slack__slack_search_public_and_private with after: set to today's date (YYYY-MM-DD). For each search use limit=10 and response_format="detailed".

1. query: "incident SEV1 SEV2 after:TODAY"
2. query: "after:TODAY in:#1st_level_customer_support"
3. query: "after:TODAY in:#2nd_level_customer_support"
4. query: "after:TODAY in:#pne-customer-support-community"
5. query: "after:TODAY in:#celostar-support"
6. query: "after:TODAY in:#cs_tools"
7. query: "AI agent automation after:TODAY"

## Step 1.5 — Enrich with CeloAssist

Identify any STILL-OPEN incidents or unresolved cases. If found, DM CeloAssist (U08M89GMBST) asking: are there known workarounds, overnight escalation paths, or similar prior incidents? What should the on-call team reference? Run sleep 45 via mcp__workspace__bash, read the thread with mcp__slack__slack_read_thread. Retry up to 2 times. Skip if no open incidents.

## Step 2 — Synthesise (EOD lens)

Focus: what needs action before log-off, what to watch tomorrow?
- 🔴 Needs Action Before Log-off: open SEVs, unanswered escalations, coverage gaps. Include "CeloAssist context:" sub-line if workarounds or escalation paths found.
- 📋 Today's Summary: resolved incidents, key decisions, process changes
- 🌅 Watch for Tomorrow: in-flight items, pending RCAs, deployments expected
- 🤖 AI / Automation Highlights: developments on AI agent work or tooling

Flag urgent items with 🔴, FYI items with 📌.
For every bullet, append → <permalink|Source> using the Permalink from the search result.

## Step 3 — Send DM

Use mcp__slack__slack_send_message to DM YOUR_SLACK_ID.
Start: "*🌆 End-of-day wrap-up — [DATE]*"
End: "_That's a wrap. Have a good evening — see you at 8 AM tomorrow. 👋_"
```

---

## Step 3 — Confirm setup to the user

Once all three tasks are created, tell the user:

"✅ **Your daily CS briefings are set up!**

- ☀️ **8 AM** — Overnight summary + CeloAssist incident enrichment
- 🕛 **12 PM** — Midday check-in + CeloAssist enrichment on new cases
- 🌆 **4 PM** — EOD wrap-up + CeloAssist workarounds for open incidents

Each briefing DMs you on Slack with a source link on every item. When active incidents are found, CeloAssist is queried automatically for historical context, prior CBE cases, and known workarounds.

**Next step:** Click **Run now** on each task in the Scheduled sidebar to pre-approve Slack permissions and preview your first briefing.

Make sure Cowork launches at login: `System Settings → General → Login Items`"
