# Job Hunt ATS Workflow (n8n)

An automated pipeline that scrapes fresh job listings daily, scores them against a resume using an LLM, tracks them in Google Sheets, and emails a ranked digest — with a one-click "Mark Applied" link.

## What it does

1. **Daily trigger (9:05 AM)** kicks off two parallel scrapers via [Apify](https://apify.com/):
   - Indeed listings (`valig~indeed-jobs-scraper`)
   - Naukri listings (`blackfalcondata~naukri-jobs-feed`)
2. **Combine & filter** — merges both sources into one batch and keeps only postings from the last 48 hours.
3. **AI scoring** — builds a prompt containing the candidate's resume summary + the job batch, sends it to **Google Gemini** (`gemini-flash-latest`), and gets back a structured JSON array: fit score (0–100), a tailored one-line pitch, ATS keyword gaps, and an auto-exclude flag for irrelevant sectors.
4. **Google Sheets tracker** — upserts each scored job into a tracker sheet, matched/deduplicated by URL.
5. **Email digest** — sends a ranked HTML summary via Gmail, each row with an "Open" link and a "Mark Applied" link.
6. **Webhook loop** — clicking "Mark Applied" hits an n8n webhook that updates the job's status back in the sheet, with an instant confirmation response.

## Architecture

```
Daily Trigger (9:05 AM)
   ├─→ Indeed Scraper (Apify) ─┐
   └─→ Naukri Scraper (Apify) ─┴─→ Combine → Build Prompt → Gemini Scoring
                                                                    │
                                                    ┌───────────────┴───────────────┐
                                                    ▼                               ▼
                                          Split → Google Sheets Tracker    Build & Send Email Digest

Mark Applied Webhook → Update Sheet Status → Respond to Webhook (confirmation)
```

## Stack

| Layer | Tool |
|---|---|
| Orchestration | n8n (self-hosted / Docker) |
| Job data source | Apify (Indeed + Naukri scrapers) |
| AI scoring | Google Gemini API (free tier) |
| Storage / tracking | Google Sheets |
| Notification | Gmail API |
| Status updates | n8n Webhook |

## Setup

This JSON is sanitized for public sharing — before importing into your own n8n instance, replace the following placeholders:

| Placeholder | Where | Replace with |
|---|---|---|
| `YOUR_APIFY_API_TOKEN` | Apify HTTP nodes | Your Apify API token |
| `YOUR_GEMINI_API_KEY` | Gemini scoring node | Your Google AI Studio API key |
| `your-email@example.com` | Gmail node | Your email |
| `YOUR_SHEET_ID` | Google Sheets nodes | Your tracker sheet's ID |
| `https://YOUR_N8N_INSTANCE...` | Email digest code node | Your n8n instance's webhook production URL |
| `YOUR_GOOGLE_SHEETS_CREDENTIAL_ID` / `YOUR_GMAIL_CREDENTIAL_ID` | Credential blocks | Re-authenticate in n8n; it will regenerate these |

**Best practice going forward:** instead of hardcoding API keys in HTTP Request node parameters, use n8n's built-in credential store (Header Auth / Generic Credential type) so keys never live in the exported JSON at all.

## Notes

- Resume/candidate details in the scoring prompt are placeholders — swap in your own summary.
- Hard-exclusion rules (e.g. sector filters) are configurable in the "Build Scoring Prompt" code node.

