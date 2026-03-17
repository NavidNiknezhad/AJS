# AJS — Automated Job Seeking Assistant

AJS is a Python CLI that converts job recommendation emails into **review-ready application packets**:

- parses recommended jobs from your inbox,
- scores job fit against your resume variants,
- selects the best resume for each posting,
- generates a tailored cover letter,
- writes a CSV shortlist and stores idempotent run history.

> **Security first:** this is a **public repository**. Never commit credentials, personal data, or private resume files.

## What this tool does

For each eligible job posting in your email folder:

1. Skips already-processed emails using SQLite idempotency.
2. Computes match score from posting text vs resume keywords.
3. Filters jobs below `JOB_BOT_MIN_MATCH_SCORE`.
4. Generates `cover_letter.md` using a template.
5. Appends the job to `generated_packets/daily_shortlist.csv`.

## Quick start

### 1) Prerequisites

- Python 3.10+
- IMAP-enabled email account (for Gmail, use App Password + IMAP enabled)

### 2) Clone and enter repo

```bash
git clone <your-fork-or-repo-url>
cd AJS
```

### 3) Create a local secrets file (do not commit)

```bash
cat > .env.local <<'ENV'
JOB_BOT_IMAP_HOST=imap.gmail.com
JOB_BOT_IMAP_USERNAME=<your-email>
JOB_BOT_IMAP_PASSWORD=<your-app-password>
JOB_BOT_EMAIL_FOLDER=INBOX
JOB_BOT_MIN_MATCH_SCORE=0.30
JOB_BOT_BASE_DIR=.
JOB_BOT_OUTPUT_DIR=generated_packets
JOB_BOT_DB=job_apply_bot.sqlite3
ENV
```

Load it for your shell session:

```bash
set -a; source .env.local; set +a
```

### 4) Create your resume manifest

Create `resume_manifest.json` (kept local/private if it contains personal paths or strategy):

```json
[
  {
    "key": "backend",
    "file_path": "resumes/backend_resume.pdf",
    "keywords": ["python", "api", "postgres", "aws", "docker"]
  },
  {
    "key": "fullstack",
    "file_path": "resumes/fullstack_resume.pdf",
    "keywords": ["typescript", "react", "next.js", "node", "graphql"]
  }
]
```

### 5) Run the bot

```bash
python -m job_apply_bot.main \
  --resume-manifest resume_manifest.json \
  --cover-letter-template job_apply_bot/templates/cover_letter.template.md
```

### 6) Review outputs

- `generated_packets/daily_shortlist.csv`
- `generated_packets/<email_id>_<resume_key>/cover_letter.md`
- `job_apply_bot.sqlite3` (processed state)

## Required information (share securely, never in Git)

If you want someone else (or an automation assistant) to configure this for you, provide the following **through a secrets manager or encrypted channel only**:

- IMAP host (usually `imap.gmail.com`)
- Email username
- Email app password
- Mail folder to scan (`INBOX`, label folder, etc.)
- Minimum match score target (e.g. `0.30`)
- Local path to resume variants (do not upload to public repo)

Recommended secure channels:

- 1Password / Bitwarden secure notes
- Doppler / AWS Secrets Manager / GCP Secret Manager
- Encrypted `.env.local` managed outside version control

## Security checklist for this public repo

- Keep `.env.local`, resumes, and generated packets out of git.
- Use app passwords, not primary mailbox password.
- Create a dedicated mailbox/account for automation when possible.
- Review shortlist manually before any application submission.
- Rotate credentials periodically and on teammate offboarding.

Recommended `.gitignore` entries:

```gitignore
.env*
resumes/
generated_packets/
*.sqlite3
resume_manifest.json
```

## Configuration reference

| Variable | Default | Description |
|---|---|---|
| `JOB_BOT_IMAP_HOST` | `imap.gmail.com` | IMAP server host |
| `JOB_BOT_IMAP_USERNAME` | `` | Email username/login |
| `JOB_BOT_IMAP_PASSWORD` | `` | Email app password |
| `JOB_BOT_EMAIL_FOLDER` | `INBOX` | Folder to scan |
| `JOB_BOT_MIN_MATCH_SCORE` | `0.30` | Filter threshold |
| `JOB_BOT_BASE_DIR` | `.` | Base path for output/db |
| `JOB_BOT_OUTPUT_DIR` | `generated_packets` | Output folder |
| `JOB_BOT_DB` | `job_apply_bot.sqlite3` | SQLite database filename |

## Operating model

- **Current stage (startup/MVP):** single-process Python monolith + SQLite + cron/manual trigger.
- **Human-in-the-loop:** generated packets are intended for review before applying.
- **Idempotent runs:** duplicate emails are skipped via `email_id` tracking.

## Short architecture review

### Scalability risks
- IMAP polling and keyword matching are linear and may degrade with very large inboxes.

### Failure points
- Email provider auth changes, malformed HTML emails, and inconsistent job links.

### Security concerns
- Credential leakage from env files, accidental commit of personal resume data, over-broad mailbox access.

### Performance bottlenecks
- Synchronous parsing/scoring for each posting in one process.

### Operational complexity
- Cron jobs lack strong observability by default; add structured logs and alerts as usage grows.

### 10x usage upgrade path
- Move to modular monolith with Postgres + Redis queue.
- Add worker pool for ingest/match/render pipeline.
- Add provider adapters (Greenhouse/Lever/Workday) and richer ranking (embeddings).

## Troubleshooting

- `Built 0 application packet(s).`
  - Check folder, score threshold, and whether emails were already processed.
- `EmailIngestionError`
  - Verify IMAP host/username/password and provider IMAP settings.
- Low-quality matching
  - Expand keywords in resume manifest and tune `JOB_BOT_MIN_MATCH_SCORE`.

## License

MIT — see [LICENSE](LICENSE).
