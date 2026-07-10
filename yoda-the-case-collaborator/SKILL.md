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

### Step 3 — Check Salesforce Files (attachments)
Use `mcp__salesforce__soqlQuery` to retrieve files attached to the case:

```
SELECT ContentDocument.Title, ContentDocument.FileExtension, ContentDocument.FileType,
       ContentDocument.ContentSize, ContentDocument.CreatedDate, ContentDocument.CreatedBy.Name,
       ContentDocument.Description
FROM ContentDocumentLink
WHERE LinkedEntityId = '<case_id>'
ORDER BY ContentDocument.CreatedDate DESC
```

List all attachments in the summary. Pay special attention to:
- **HAR files** (`.har`) — if present, note the filename and offer to analyse it with the `har-analyzer` skill.
- **Audit logs / export files** — `.log`, `.txt`, `.csv`, `.zip` files that may contain diagnostic data.
- **Screenshots** — `.png`, `.jpg` files showing UI errors.

If a HAR file is found, proactively tell the user: _"I found a HAR file attached ([filename]). Would you like me to run the HAR analyser on it?"_

If no files are attached, note this explicitly — it may be worth asking the customer to upload logs.

### Step 4 — Search Jira
Use `mcp__atlassian__search` with query: `case <case_number> <first_word_of_account_name>`

Extract any Jira issues from the results (items where `type = "issue"` or the URL contains `/browse/`).

### Step 5 — Search Slack
Use `mcp__slack__slack_search_public` with `query: "<case_number>"` and `limit: 8`.

Collect channel name, author, message text snippet, timestamp, and permalink.

### Step 6 — Search docs.celonis.com for known issues and guidance
Extract 3–5 keywords from the case `Subject` and `Description` (e.g. product area, error terms, feature names). Then use `mcp__workspace__web_fetch` to search Celonis documentation:

```
https://docs.celonis.com/en/search.html?q=<url-encoded-keywords>
```

If the subject points to a specific feature area, also fetch the relevant section directly, for example:
- Permissions / user access: `https://docs.celonis.com/en/permissions-and-access-management.html`
- Task Mining: `https://docs.celonis.com/en/task-mining.html`
- Data integration / extraction: `https://docs.celonis.com/en/data-integration.html`
- Process Analytics / Studio: `https://docs.celonis.com/en/process-analytics.html`

Extract 2–4 relevant findings and classify each as:
- 🟡 **Known product limitation** — behaviour is by design
- 🔵 **Configuration issue** — customer is missing a setup step or has wrong settings
- 🟢 **Documented fix / workaround** — resolution steps exist in the docs
- 🔴 **No documentation found** — potential product bug; consider raising a Jira

Always include direct links to the relevant doc pages so they can be copied into the case reply.

### Step 7 — Summarise and recommend
**Always begin the output with this exact title format:**
```
# Case <case_number> — <Account.Name>
```
For example: `# Case 00439352 — Acme Corporation`

Then present the full structured summary:

1. **Case snapshot** — subject, status, priority, account, owner, age, last update
2. **Email thread** — chronological list of emails: who sent it, when, and a 2-line summary of content. Flag if there has been no response in >48h.
3. **Attached files** — list all files with filename, type, size, uploaded by, and date. Call out HAR files and diagnostic logs explicitly.
4. **Docs & known issues** — findings from docs.celonis.com, classified by type. Include direct doc links.
5. **Jira tickets** — list any linked issues with key, summary, and status. If none found, say so explicitly.
6. **Slack mentions** — list relevant threads with channel and a snippet. Link to Slack directly.
7. **Next actions** — 3–5 concrete, specific next steps to drive the case to closure. Consider:
   - Is the case waiting on the customer? Say so and suggest a follow-up template.
   - Is it waiting on PnE/engineering? Suggest escalation path.
   - Is there a Jira ticket that needs an update?
   - Has the SLA been breached or is it at risk?
   - Does the docs search suggest this can be closed as a known limitation or config fix?

## Output format
**Every response must start with `# Case XXXXXXXX — Customer Name` as the title.** Write in prose with clear section headers. Do not use excessive bullet nesting. Keep it scannable — this is for an L2 engineer triaging in real time.

## Important notes
- This skill is used by the **L2 Horizon team** at Celonis EMEA (AAM and Task Mining products).
- The engineer is a Senior L2 CSE — assume technical depth, skip basics.
- Always check case age vs priority — a Critical case open for >1 day without update should be flagged prominently.
- GDPR: do not repeat full customer email addresses or PII in your summary beyond what is necessary. Refer to contacts by name only.
- If the user says "draft a reply", use the email thread context and doc findings to write a professional, empathetic Celonis-style customer response. Include relevant doc links in the reply where useful.
- If a HAR file is present, always recommend running the `har-analyzer` skill before drafting a reply.
- If docs.celonis.com confirms a known limitation, flag this clearly — it changes how the case is closed (explanation to customer vs. awaiting a product fix).
