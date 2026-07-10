---
name: "yoda-the-case-collaborator"
description: "Yoda — The Case Collaborator. L2 Horizon skill: given a Salesforce case number, pulls case details, email thread, linked Jira tickets, and Slack mentions, then suggests next actions to drive the case to closure."
---

# Yoda — The Case Collaborator

## When to trigger
Invoke this skill when the user provides a Salesforce case number (format: 8-digit number like `00439828`) and asks to:
- "open the case", "look up case XXXXXXXX", "what's the status of case..."
- "collaborate on a case", "help me close this case"
- or anything involving reviewing a Celonis support case

## What to do

### Step 1 — Fetch Salesforce case
Use `mcp__salesforce__soqlQuery` to retrieve the case:

```
SELECT Id, CaseNumber, Subject, Status, Priority, Account.Name, Type, Origin,
       Owner.Name, CreatedDate, LastModifiedDate, ClosedDate, Reason, Description
FROM Case WHERE CaseNumber = '<case_number>'
```

If no record is found, tell the user clearly and stop.

### Step 2 — Fetch email thread
Use the returned `Id` to query Tasks (which hold the email history):

```
SELECT Id, Subject, Description, ActivityDate, Status, CreatedDate, Owner.Name
FROM Task WHERE WhatId = '<case_id>'
ORDER BY CreatedDate DESC LIMIT 20
```

The `Description` field contains the full email body. Extract the content after `Body:` and before the `----` separator line.

### Step 3 — Search Jira
Use `mcp__atlassian__search` with query: `case <case_number> <first_word_of_account_name>`

Extract any Jira issues from the results (items where `type = "issue"` or the URL contains `/browse/`).

### Step 4 — Search Slack
Use `mcp__slack__slack_search_public` with `query: "<case_number>"` and `limit: 8`.

Collect channel name, author, message text snippet, timestamp, and permalink.

### Step 5 — Summarise and recommend
Present a structured summary:

1. **Case snapshot** — subject, status, priority, account, owner, age, last update
2. **Email thread** — chronological list of emails: who sent it, when, and a 2-line summary of content. Flag if there has been no response in >48h.
3. **Jira tickets** — list any linked issues with key, summary, and status. If none found, say so explicitly.
4. **Slack mentions** — list relevant threads with channel and a snippet. Link to Slack directly.
5. **Next actions** — 3–5 concrete, specific next steps to drive the case to closure. Consider:
   - Is the case waiting on the customer? Say so and suggest a follow-up template.
   - Is it waiting on PnE/engineering? Suggest escalation path.
   - Is there a Jira ticket that needs an update?
   - Has the SLA been breached or is it at risk?

## Output format
Write in prose with clear section headers. Do not use excessive bullet nesting. Keep it scannable — this is for an L2 engineer triaging in real time.

## Important notes
- This skill is used by the **L2 Horizon team** at Celonis EMEA (AAM and Task Mining products).
- The engineer is a Senior L2 CSE — assume technical depth, skip basics.
- Always check case age vs priority — a Critical case open for >1 day without update should be flagged prominently.
- GDPR: do not repeat full customer email addresses or PII in your summary beyond what is necessary. Refer to contacts by name only.
- If the user says "draft a reply", use the email thread context to write a professional, empathetic Celonis-style customer response.
