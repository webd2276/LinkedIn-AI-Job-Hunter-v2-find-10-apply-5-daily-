# LinkedIn AI Job Hunter v2

An n8n workflow that runs every day, reads your CV, finds new LinkedIn jobs that match it, scores each job with AI, writes a cover letter for each one, and tries to apply to the best matches with LinkedIn Easy Apply through a cloud browser. Every job goes into a Google Sheet, and you get an email for each result.

> **Default limits:** it finds up to **10** new jobs a day and applies to up to **5** (only jobs scoring at least 70/100).

---

## Features

- **CV-based search**: an AI model reads your CV (PDF on Google Drive) and picks your field, seniority, skills, top job titles and search keywords.
- **Daily LinkedIn job search**: uses an Apify LinkedIn Jobs scraper and only looks at jobs posted in the last 24 hours.
- **No duplicates**: skips jobs that are already in your Google Sheet (it checks the URL and the company + title pair).
- **Filters**: leave out companies or title words you don't want (for example `intern`, `unpaid`, `volunteer`).
- **AI scoring and cover letters**: each job gets a 0–100 match score, a one-line reason and a cover letter of about 120 words based only on what is in your CV.
- **Automatic Easy Apply**: a Browserbase cloud browser logs in to LinkedIn and submits Easy Apply forms. If it hits a captcha, a verification check, or a required question it can't answer truthfully from your CV, it stops and does not submit.
- **Dry-run mode**: test the whole flow without applying to anything.
- **Retry queue**: jobs marked `queued` or `error` are tried again on the next run.
- **Google Sheets log**: one row per job with its score, status, reason, note and date.
- **Email alerts**: Gmail sends you an email for each job (applied, review and apply, or apply manually), with the cover letter included.
- **Error alerts**: an Error Trigger emails you if the workflow fails.

---

## How it works

```
Daily 9AM Trigger
  -> Config
  -> Download CV (Google Drive)
  -> Extract CV Text (PDF)
  -> Analyze CV (AI)             -> Parse Profile
  -> Search LinkedIn Jobs (Apify)
  -> Read Jobs Log (Google Sheets)
  -> Pick New Jobs (dedupe + filters + re-queue)
  -> No New Jobs? --yes--> Email "no new jobs"
                 --no---> Score + Cover Letter (AI) -> Parse Scores + Rank
                            |-> Log Found Jobs (Google Sheets)
                            |-> Filter To Apply (top N >= min_score)
                                  -> Dry Run? --yes--> Mark Dry Run -> Email
                                              --no---> Apply via Browser (Browserbase)
                                                         -> Parse Apply Results
                                                              |-> Update Status (Sheets)
                                                              |-> Email

Error Trigger -> Email alert
```

---

## Tech stack

| Purpose | Service |
|---|---|
| Automation | [n8n](https://n8n.io) |
| CV storage | Google Drive |
| AI (CV analysis, scoring, cover letters) | [Pollinations](https://pollinations.ai) API (OpenAI-compatible) |
| Job search | [Apify](https://apify.com) `curious_coder/linkedin-jobs-scraper` |
| Job log | Google Sheets |
| Auto-apply | [Browserbase](https://browserbase.com) n8n node |
| Notifications | Gmail |

---

## Setup

### 1. Import the workflow
In n8n, go to **Workflows > Import from file** and pick `workflow.json`.

### 2. Add credentials
| Credential | Used by | Notes |
|---|---|---|
| Google Drive OAuth2 | Download CV | |
| Google Sheets OAuth2 | Read Jobs Log, Log Found Jobs, Update Status | |
| Gmail OAuth2 | All email nodes | |
| Header Auth | Both Pollinations nodes | Name: `Authorization`, Value: `Bearer <your_pollinations_key>` |
| Query Auth | Search LinkedIn Jobs | Name: `token`, Value: `<your_apify_token>` |
| Browserbase API | Apply via Browser | |

### 3. Create the Google Sheet
Make a sheet with a tab named `jobs` and this header row:

```
job_url | title | company | score | status | reason | note | date
```

### 4. Edit the `Config` node

| Field | Description | Default |
|---|---|---|
| `cv_file_id` | Google Drive file ID of your CV (PDF) | — |
| `sheet_id` | Google Sheet ID | — |
| `find_limit` | Most new jobs to find per run | `10` |
| `apply_limit` | Most applications per run | `5` |
| `min_score` | Lowest AI match score needed to apply | `70` |
| `dry_run` | `true` = don't apply, just email the matches | `false` |
| `phone` | Phone number used in Easy Apply forms | — |
| `exclude_companies` | Comma-separated companies to skip | empty |
| `exclude_title_words` | Comma-separated title words to skip | `intern,unpaid,volunteer` |

> Also put your Sheet ID in the **Read Jobs Log**, **Log Found Jobs** and **Update Status** nodes, your CV file ID in **Download CV**, and your email address in the Gmail nodes.

### 5. Add your LinkedIn login
In **Apply via Browser (Browserbase)**, set the `li_email` and `li_password` variables. **Never commit real values** (see the Security section).

### 6. Turn on error alerts
Go to **Workflow Settings > Error workflow** and select this same workflow.

### 7. Test, then publish
1. Set `dry_run = true` and run the workflow by hand.
2. Check the Google Sheet and the emails.
3. Set `dry_run = false` and publish the workflow.

---

## Job statuses

| Status | Meaning |
|---|---|
| `found` | Logged but not chosen for applying |
| `below_threshold` | Score was below `min_score` |
| `queued` | Chosen for applying (tried again next run if not finished) |
| `dry_run` | Dry-run mode, sent to you to review |
| `applied` | Easy Apply application submitted |
| `manual_needed` | Skipped (no Easy Apply, captcha, verification, or a question it couldn't answer); apply by hand |
| `error` | Browser or automation error; tried again next run |

---

## Security

- **Never commit secrets.** Before you export the workflow to GitHub, remove the LinkedIn password, API keys, phone number and personal IDs. n8n credentials are not included in exports, but values typed straight into node fields (like Browserbase variables) **are**.
- The AI prompts treat CV and job text as untrusted data, which helps protect against prompt injection.
- The bot never makes up experience, degrees or visa status. If a required question can't be answered truthfully, it skips the job.

---

## Disclaimer

Automating LinkedIn may break LinkedIn's [User Agreement](https://www.linkedin.com/legal/user-agreement) and can get your account restricted. Use this project at your own risk, keep the daily limits low, and check the applications it sends. This project is for learning purposes.

---

## License

MIT

![Alt Text](https://github.com/webd2276/LinkedIn-AI-Job-Hunter-v2-find-10-apply-5-daily-/blob/main/json.png)

