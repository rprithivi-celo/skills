---
name: "shift-handover"
description: "Generate a structured L2 shift handover from case IDs and status notes, then post to #l2-shift-handover Slack channel."
---

# Shift Handover Generator

## Purpose
Generate a structured, professional L2 Horizon shift handover from minimal input (case IDs + one-line status per case), then post it to the `#l2-shift-handover` Slack channel (channel ID: `C0BCXL7U1QF`).

## Trigger
User pastes case IDs + status notes and asks to generate a handover, OR says something like "create handover", "shift handover", "end of shift", "handover time".

## Step 1 — Parse the input
Extract for each case:
- Case ID / Salesforce URL (if provided)
- CBE / Jira link (if provided)
- Customer name
- Product area (Data Platform, Process Mining, APM, OCPM, Task Mining, AAM, etc.)
- Severity (Sev-1 through Sev-4)
- One-line issue description
- Current status / blockers
- Actions needed for the next shift

If any of these are missing and the case is Sev-1 or Sev-2, ask the user to fill in the gaps before generating. For Sev-3/4, use "Unknown" for missing fields and proceed.

## Step 2 — Generate the handover

Use the following format. Generate one block per case.

---

**Shift Handover — [Date]**
*L2 Horizon — [Your name if known, otherwise omit]*

---

**Case [N]: [Case ID] — [Customer Name] ([Issue Summary])**
- **Salesforce:** [URL or "Case ID: XXXXXXXX"]
- **CBE/Jira:** [link or "None raised"]
- **Product Area:** [area]
- **Severity:** SEV-[N] [— add brief business context if Sev-1 or Sev-2, e.g. "€2M p.a. use case at risk"]

**Issue:**
[2-3 sentence description of what the customer is experiencing and what has been tried]

**Current Status:**
- Waiting on Customer: [what was requested, or "Nothing pending"]
- Waiting on PnE: [CBE status / what was pinged, or "No CBE raised"]

**Actions for Next Shift:**
- [Specific, actionable next steps — not vague. E.g. "If HAR file arrives → upload to CBE-54442 and tag @max"]

---

Repeat for each case. End with:

**CC:** @[relevant team members who are tagged on these cases]

---

## Step 3 — Confirm before posting
Show the generated handover to the user in chat and say:
> "Ready to post to #l2-shift-handover. Confirm with 'send it' or edit anything first."

Do NOT post until the user explicitly confirms.

## Step 4 — Post to Slack
Once confirmed, post to channel ID `C0BCXL7U1QF` (`#l2-shift-handover`) using `slack_send_message`.

Use Slack markdown:
- `*bold*` for headers/labels
- Bullet points with `-`
- Links as `<URL|display text>`
- Mention users as `<@USERID>` if their Slack IDs are known

## Format guidance (from real examples)

**L2 Engineer's style** (reference for quality):
- Title includes customer name, issue type, and date
- Explicit "Waiting on Customer" / "Waiting on PnE" sections
- Actions Needed are conditional and specific (if X happens → do Y)
- Includes business context (revenue at risk, use case criticality)

**L2 Engineer 2's style** (reference for brevity when multiple cases):
- Compact numbered list when handing over 3+ cases quickly
- CBE link + product area + severity inline
- CC relevant team members at the bottom

**Default to L2 Engineer's format** for 1-2 cases. Switch to L2 Engineer 2's compact style if the user provides 3+ cases with minimal detail.

## Tone
Professional but direct. This is an internal team communication — no fluff, no excessive hedging. The next engineer reading this needs to be able to act without asking questions.

## GDPR note
Do not include customer PII (names, emails, phone numbers) beyond what is already in the case ID or Salesforce link. If the user pastes raw customer data, strip it before posting to Slack.
