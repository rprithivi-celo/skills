# CS Daily Briefing — Cowork Skill

A Cowork skill that sets up **3 automated daily Slack briefings** for Celonis Customer Support engineers. Scans key support and incident channels, enriches active incidents using CeloAssist, and DMs you a structured summary with one-click source links — three times a day.

| Time | Briefing | Focus |
|------|----------|-------|
| ☀️ 8:00 AM | Morning | Overnight catch-up + CeloAssist incident enrichment |
| 🕛 12:00 PM | Midday | New activity since morning |
| 🌆 4:00 PM | End of Day | What needs action before log-off + tomorrow's watch list |

---

## Prerequisites

- [Claude Cowork](https://claude.ai/download) desktop app installed
- **Slack plugin** connected in Cowork
- Cowork set to **launch at login**: `System Settings → General → Login Items`

---

## Installation

### Option A — One-click install (recommended)

1. Download `cs-daily-briefing.skill` from the [Releases](../../releases) page
2. Double-click the file — Cowork will prompt you to install it
3. Once installed, open Cowork and type: `/cs-daily-briefing`

### Option B — Manual install

1. Clone this repo
2. In Cowork, open Settings → Skills → Install from folder
3. Point it at the `cs-daily-briefing-skill/` directory

---

## Usage

Once installed, run the skill by typing in Cowork:

```
/cs-daily-briefing
```

The skill will:
1. Auto-detect your Slack user ID
2. Create (or update) the 3 scheduled tasks
3. Confirm setup with a summary

**After setup:** Click **Run now** on each task in the Scheduled sidebar to pre-approve Slack permissions and preview your first briefing.

---

## Channels monitored

- `#1st_level_customer_support`
- `#2nd_level_customer_support`
- `#pne-customer-support-community`
- `#celostar-support`
- `#cs_tools`
- Any active `#incident-*` channels
- AI / automation discussions across workspace

---

## CeloAssist enrichment

When active SEV1/SEV2 incidents are detected, the briefing automatically queries **CeloAssist** for historical context, prior CBE cases, and known workarounds — adding a `CeloAssist context:` sub-line to each incident entry.

---

## Customisation

To change channels, timing, or summary format, tell Cowork:

> "Update my daily briefing to also scan #my-channel"
> "Change my morning briefing to 7:30 AM"

Or edit the scheduled task prompts directly from the Scheduled sidebar.

---

## Uninstall

To remove the scheduled tasks:
1. Open the Scheduled sidebar in Cowork
2. Delete `daily-slack-cs-briefing`, `midday-slack-cs-briefing`, and `eod-slack-cs-briefing`

---

## Built with

- [Claude Cowork](https://claude.ai) — AI desktop automation
- Slack MCP plugin
- CeloAssist (internal Celonis AI assistant)
