# AI Lead Enrichment & Outreach Pipeline

> Turn a raw list of leads into researched, personalized, human-approved outreach emails, automatically.

Built with **n8n · Serper.dev · Groq (Llama 3.3) · Google Sheets · Gmail API · Telegram**

---

## The Problem

Sales and outreach teams lose hours on the same repetitive loop:

1. Look up each lead on Google
2. Work out what the company actually does
3. Write a personalized email from scratch
4. Copy, paste, and send it

Generic mass emails get ignored. Personalized ones take too long. This pipeline automates the research and writing, and keeps a human in control of what gets sent.

## The Solution

Add a lead to a Google Sheet and the pipeline does the rest:

- Searches the web for the lead's company
- Summarizes what it finds with AI
- Drafts a personalized outreach email
- Sends the draft to Telegram for approval
- Sends the email through Gmail only after you tap **Approve**

**Nothing is sent without human approval.** Draft-for-review is the default, not an option.

---

## Architecture

The pipeline is two connected flows in one n8n workflow file.

### Flow 1: Enrichment

```
Google Sheets Trigger (new lead row)
        ↓
Filter (valid lead check)
        ↓
Serper.dev web search
        ↓
Check search success → Groq: summarize company research
        ↓
Check summary success → Groq: draft personalized email
        ↓
Check draft success → Google Sheets update (save summary + draft)
        ↓
Telegram approval prompt (Approve / Reject buttons)
```

### Flow 2: Approval

```
Telegram trigger (button tap)
        ↓
Parse callback (action + contact_email)
        ↓
Route Approve or Reject
   ├─ Approve → Google Sheets lookup → Gmail send → status updated
   └─ Reject  → Google Sheets status update → "rejected"
```

### Error Handling

- **Continue On Fail** is enabled on the search and AI steps
- Success checks catch empty outputs from search or AI
- Failures route to a **Telegram alert**, so a bad lead never silently breaks the run

---

## Tech Stack

| Layer | Tool | Role |
|---|---|---|
| Automation engine | n8n (self-hosted) | Workflow orchestration |
| Lead intake | Google Sheets | Lead database and status tracking |
| Web research | Serper.dev | Google search results API |
| AI | Groq API (Llama 3.3) | Summarization and email drafting |
| Approval UI | Telegram Bot API | Inline Approve / Reject buttons |
| Outreach | Gmail API (OAuth) | Sending approved emails |
| Tunnel | Cloudflare Tunnel | Exposes local n8n for Telegram callbacks |

---

## Google Sheet Structure

Create a sheet named `Lead Pipeline` (tab `Sheet1`) with these columns:

| Column | Purpose |
|---|---|
| `lead_id` | Unique ID for the lead |
| `company_name` | Company to research |
| `contact_name` | Person you are writing to |
| `contact_email` | Contact email (used as the lookup key) |
| `website` | Company website |
| `notes` | AI-generated company summary is saved here |
| `status` | Tracks the lead: pending, then sent or rejected |
| `date_added` | When the lead was added |
| `email_draft` | AI-drafted email |

> Column headers are referenced in n8n expressions. Names must match exactly, including capitalization.

---

## Setup Guide

### Prerequisites

- n8n running locally (self-hosted)
- Cloudflare Tunnel (`cloudflared`) installed
- Accounts: Google Cloud, Serper.dev, Groq, Telegram (via @BotFather)

### 1. Import the workflow

1. In n8n, go to **Workflows → Import from file**
2. Select `lead-enrichment-pipeline.json`

### 2. Replace the placeholders

The exported file has secrets and IDs removed. Search the workflow for these and fill in your own:

| Placeholder | Where to get it |
|---|---|
| `YOUR_SERPER_API_KEY` | Your Serper.dev dashboard (in the Serper Search node header) |
| `YOUR_TELEGRAM_CHAT_ID` | Your Telegram chat ID (in the Telegram nodes) |
| `YOUR_GOOGLE_SHEET_ID` | The ID in your Google Sheet URL (in the Google Sheets nodes) |

### 3. Add credentials in n8n

| Credential | What you need |
|---|---|
| Google Sheets OAuth2 | Client ID and Client Secret from a Google Cloud project with the Google Sheets and Google Drive APIs enabled |
| Gmail OAuth2 | Same Google Cloud project, with the Gmail API enabled and your sending account added as a test user |
| Groq | API key from the Groq console |
| Telegram | Bot token from @BotFather |

### 4. Connect Telegram

1. Message your bot once so it can reply to you
2. Put your chat ID into the Telegram nodes
3. Make sure n8n is reachable through your Cloudflare Tunnel URL so button taps reach the approval flow

### 5. Activate

Publish the workflow in n8n. It must be active to receive Approve / Reject taps.

---

## Usage

1. Add a lead row to the Google Sheet and set `status` to `pending`
2. Wait for the Telegram message with the draft email
3. Tap **Approve** to send through Gmail, or **Reject** to discard
4. Check the `status` column for the result

---

## Limitations

- **Needs n8n running.** The pipeline only works while your n8n instance and tunnel are online
- **Tunnel URL.** Quick Cloudflare tunnels can change address on restart, which breaks Telegram callbacks until the webhook is updated. A named tunnel avoids this
- **Free-tier limits.** Serper.dev and Groq limit usage on free plans, so large batches may need pacing
- **Gmail limits.** Sending is subject to Gmail's daily limits, and cold email can still land in spam
- **AI drafts need review.** The summary comes from search snippets, so check facts before approving
- **Contact emails are yours to supply.** The pipeline does not find email addresses

---

## Responsible Use

This tool drafts outreach for human review. Only email people you have a legitimate reason to contact, follow the email and privacy laws that apply to your recipients (for example CAN-SPAM and GDPR), and include a way to opt out.

---

## Author

**Isaac Desmond** · AI Automation & Full-Stack Developer

- Portfolio: [codexwayde.github.io/portfolio](https://codexwayde.github.io/portfolio)
- GitHub: [@CodexWayde](https://github.com/CodexWayde)
- LinkedIn: [isaac-desmond](https://www.linkedin.com/in/isaac-desmond-b23a213a7)
