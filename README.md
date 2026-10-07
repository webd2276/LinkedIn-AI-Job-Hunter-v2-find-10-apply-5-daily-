# LinkedIn AI Job Hunter

An n8n workflow that reads your CV, finds fresh LinkedIn jobs every day, scores each one against your profile with AI, writes a tailored cover letter, and queues the best matches. A small Playwright bot running on **your own computer** then applies to the queued jobs through LinkedIn Easy Apply.

**Daily target:** find 10 new jobs, apply to the top 5.

> **Read this first.** Automating actions on LinkedIn is against LinkedIn's User Agreement and can get your account restricted or banned. This project is for personal learning and use at your own risk. The bot is deliberately slow and cautious, but it cannot remove that risk. Start with the manual-review mode (see [Modes](#modes)).

---

## How it works

```
 n8n (cloud or self-hosted)                                  Your computer
┌──────────────────────────────────────────────────┐        ┌──────────────────────────┐
│ 09:00 trigger                                    │        │  apply-linkedin.js       │
│   -> read CV from Google Drive, extract text     │        │  (Playwright, your       │
│   -> AI analyses CV: field, titles, keywords     │        │   logged-in browser)     │
│   -> Apify LinkedIn scraper: jobs from last 24h  │        │                          │
│   -> drop duplicates already in Google Sheet     │        │  1. GET  /job-queue  ────┼──┐
│   -> keep 10 new jobs                            │        │  2. apply via Easy Apply │  │
│   -> AI scores each job (0-100) + cover letter   │        │  3. POST /job-result ────┼──┤
│   -> log all jobs to Google Sheet                │        └──────────────────────────┘  │
│   -> top 5 above min score get status "queued"   │                                      │
│   -> email you the top matches                   │  <───────── webhooks (x-bot-key) ────┘
└──────────────────────────────────────────────────┘
```

The Google Sheet is the single source of truth. Every job ends up with a status: `found`, `below_threshold`, `queued`, `applied`, `manual_needed` or `error`.

## Features

- **CV-driven search.** Job titles, keywords and location come from your CV, not hard-coded values.
- **Strict AI scoring.** Each job gets a 0-100 match score and a one-line reason. Only jobs at or above `min_score` are queued.
- **Honest cover letters.** The prompt tells the model to claim only skills that actually appear in your CV.
- **Duplicate protection.** Jobs are matched by URL and by company + title, so you never see or apply to the same job twice.
- **Safe skipping.** If an application asks a question the bot does not know the answer to, it discards the form and marks the job `manual_needed` instead of guessing.
- **Production touches.** Retries on every network node, an error workflow that emails you on failure, a config node for all settings, and a "no new jobs" notice.
- **Free AI option.** Uses [Pollinations](https://enter.pollinations.ai/) by default, no paid LLM account needed.

## Stack

| Part | Used for |
|---|---|
| [n8n](https://n8n.io) | Orchestration (works on n8n Cloud) |
| [Pollinations](https://enter.pollinations.ai/) | CV analysis, job scoring, cover letters |
| [Apify](https://apify.com) LinkedIn jobs scraper | Fetching fresh jobs |
| Google Drive + Google Sheets | CV storage and job log / queue |
| Gmail | Notifications and error alerts |
| [Playwright](https://playwright.dev) (Node.js) | Local Easy Apply bot |

## Repository layout

```
.
├── workflow/
│   └── linkedin-job-hunter.n8n.json   # import this into n8n
├── bot/
│   ├── apply-linkedin.js              # local Playwright bot
│   └── .env.example                   # copy to .env and fill in
├── .gitignore
└── README.md
```

## Setup

### 1. Google Sheet

Create a sheet with a tab named `Jobs` and this header row:

```
job_url | title | company | score | status | reason | note | date | cover_letter
```

Upload your CV (PDF) to Google Drive and note its file ID.

### 2. n8n credentials

| Credential | Settings |
|---|---|
| Google Drive, Google Sheets, Gmail | Standard OAuth2 connection in n8n |
| **Pollinations** (Header Auth) | Name: `Authorization`  Value: `Bearer YOUR_POLLINATIONS_KEY` |
| **Apify** (Query Auth) | Name: `token`  Value: your Apify API token |
| **Bot Key** (Header Auth) | Name: `x-bot-key`  Value: a long random secret (`openssl rand -hex 24`) |

> The Header Auth **Name** is the HTTP header name, so it must be exactly `Authorization` (no spaces). Putting `Bearer ...` or "bearer token" there causes `Header name must be a valid HTTP token`.

### 3. Import and configure the workflow

1. In n8n choose **Import from file** and select `workflow/linkedin-job-hunter.n8n.json`.
2. Open the **Config** node and set: CV file ID, Sheet ID, `notify_email`, `find_limit` (10), `apply_limit` (5), `min_score` (70).
3. In **Search LinkedIn Jobs** replace `YOUR_LINKEDIN_JOBS_ACTOR_ID` with your Apify actor (`username~actor-name`). If your actor uses different input or output field names, adjust the request body and the **Pick New Jobs** node.
4. Select your credentials on every node that shows a warning (Google, Gmail, Pollinations, Apify, and Bot Key on both webhook nodes).
5. Replace `YOUR_SHEET_ID` and `YOUR_EMAIL@gmail.com` where they still appear.
6. In the workflow settings set **Error workflow** to this same workflow so the failure email works.
7. Run it once manually, check the Sheet, then switch the workflow to **Active**.
8. Open **Get Queue Webhook** and **Result Webhook** and copy their **Production URL** (it only works while the workflow is active).

### 4. Local bot

Requires Node.js 18 or newer.

```bash
cd bot
npm i playwright
npx playwright install chromium
cp .env.example .env      # then edit the values
node apply-linkedin.js --login   # log in to LinkedIn by hand, then close the window
node apply-linkedin.js --queue   # fetch queued jobs and apply
```

`.env`:

```
QUEUE_URL=https://YOUR-N8N/webhook/job-queue
RESULT_URL=https://YOUR-N8N/webhook/job-result
BOT_KEY=same-secret-as-in-the-n8n-Bot-Key-credential
PHONE=+92XXXXXXXXXX
CV_PATH=/absolute/path/to/cv.pdf
MAX_APPLY=2
```

Start with `MAX_APPLY=2` or `3` and watch the browser. Raise it to 5 once it behaves.

On Linux distributions Playwright does not officially support, the install prints a "BEWARE" notice and uses a fallback build. If Chromium reports missing libraries, install them with your package manager.

### 5. Run it daily

```cron
30 9 * * * cd /path/to/bot && /usr/bin/node apply-linkedin.js --queue >> bot.log 2>&1
```

Your computer must be on and logged in to a desktop session, because the browser opens visibly. On Windows use Task Scheduler.

## Modes

| Mode | What happens | When to use |
|---|---|---|
| **Manual review** | Do not run the bot. The workflow emails you the top jobs with the link and cover letter, and you apply yourself. | Recommended starting point, zero account risk |
| **Bot apply** | Run `apply-linkedin.js --queue`. The bot applies and reports back to the Sheet. | After you trust the scoring and have tested the bot |

## Safety measures built into the bot

- Maximum applications per run (`MAX_APPLY`), default 5.
- Random 45-120 second pause between applications.
- Persistent real browser profile, you log in yourself. No password is ever stored by the script.
- Unknown required question: the application is discarded and the job is marked `manual_needed`.
- Webhooks require the `x-bot-key` header.

Still recommended: ramp up gradually, never run it on a brand-new account, and stop if LinkedIn shows a warning or captcha.

## Troubleshooting

| Problem | Fix |
|---|---|
| `Header name must be a valid HTTP token` | Header Auth **Name** must be `Authorization`, value must start with `Bearer ` |
| `Authorization failed - check your credentials` (Pollinations) | Value needs the `Bearer ` prefix, use an `sk_` secret key, no stray spaces |
| Apify `402 ... Request not authenticated` | Attach the Apify Query Auth credential (`token`) to the search node |
| Apify `datePosted must be equal to one of the allowed values` | Use the exact value your actor accepts (for example `r86400` or `past24Hours`) |
| `ENOENT ... /opt/job-bot` | Old self-hosted file-write nodes were used on n8n Cloud. Use the current workflow |
| "No new jobs" but jobs exist | Field names in **Pick New Jobs** do not match your Apify actor's output |
| Bot gets `403` from n8n | `BOT_KEY` does not match the Bot Key credential, or you used the Test URL instead of the Production URL |
| Bot says `Jobs to process: 0` | Nothing has status `queued` in the Sheet yet. Run the workflow first |
| Easy Apply button not found | The job applies on the company site. It is marked `manual_needed` |

## Security notes

- Never commit `.env`, API keys, or the `li-profile/` folder. The profile folder contains your logged-in LinkedIn session.
- If a key is ever pasted into a chat, issue, or commit, revoke it and create a new one.
- Keep the Bot Key secret. Anyone with it can read your queued jobs and cover letters.

## Limitations

- Only LinkedIn **Easy Apply** jobs can be applied to automatically.
- LinkedIn changes its page markup often, so selectors in the bot may need updates.
- The Pollinations free tier can be slower and less consistent than paid models. Always skim the cover letters.
- The workflow has not been tested against LinkedIn's live site in every region and account type.

## Disclaimer

This project is not affiliated with or endorsed by LinkedIn. You are responsible for complying with LinkedIn's terms and with the laws that apply to you. The authors accept no liability for restricted accounts or missed opportunities.

## License

MIT
![Alt Text](https://github.com/webd2276/LinkedIn-AI-Job-Hunter-v2-find-10-apply-5-daily-/blob/main/json.png)

