# API Design Intelligence

> **Transform meeting transcripts into production-ready API specifications using Claude AI**

Built by **Rahul Chadha** | [GitHub](https://github.com/rchadh) | [Live Demo](https://rchadh.github.io/api-analyzer)

---

## What is Meridian?

Meridian is an AI-powered tool that analyzes meeting transcripts between business stakeholders and API designers, and automatically extracts:

- **REST API Endpoints** — HTTP methods, paths, and parameters
- **Business Rules** — validation constraints and domain logic
- **Ambiguities** — gaps and unclear requirements that need follow-up
- **OpenAPI 3.0 YAML** — a ready-to-use specification for Swagger, Postman, or any API gateway

The goal is to reduce time-to-market for API design by eliminating the manual, error-prone translation step between *what was discussed* and *what gets built*.

---

## Live Demo

**[https://rchadh.github.io/api-analyzer](https://rchadh.github.io/api-analyzer)**

No login required. No API key needed. Just paste a transcript and click **Analyze**.

---

## Architecture

```
Browser (GitHub Pages)
        │
        │  POST /api/analyze
        ▼
Cloudflare Worker  ──────────────────────► Anthropic Claude API
  (Secure Proxy)        API Key stored          (claude-haiku)
                        as encrypted
                        Worker secret
```

The architecture uses three free-tier services:

| Layer | Service | Purpose |
|-------|---------|---------|
| Frontend | GitHub Pages | Hosts the HTML/JS UI |
| Proxy | Cloudflare Worker | Securely forwards requests, holds the API key |
| AI | Anthropic Claude API | Analyzes transcripts and generates specs |

The Cloudflare Worker acts as a reverse proxy so the Anthropic API key is **never exposed** to the browser or end users.

---

## How It Works

### Step 1 — User pastes a meeting transcript

The frontend accepts raw transcript text — any format works: Zoom transcripts, Teams exports, copy-pasted meeting notes, or manually written summaries.

### Step 2 — Two sequential Claude API calls

To avoid JSON truncation errors, the analysis is split into two focused calls:

**Call 1 — Extraction** (`max_tokens: 2500`)
```
System: Extract API design information. Return JSON only.
Output: { endpoints[], businessRules[], ambiguities[] }
```

**Call 2 — YAML Generation** (`max_tokens: 2500`)
```
System: Generate OpenAPI 3.0 YAML. Return raw YAML only.
Input:  Extracted endpoints + business rules from Call 1
Output: Complete openapi: "3.0.0" specification
```

### Step 3 — Results displayed across four tabs

| Tab | Content |
|-----|---------|
| Endpoints | HTTP method, path, description, parameters |
| Business Logic | Validation rules and domain constraints |
| Ambiguities | Gaps flagged for follow-up |
| OpenAPI YAML | Copy-paste ready specification |

---

## Sample Transcript

Try pasting this into the tool:

```
PM: We need an endpoint to fetch all orders for a customer.
Dev: Should we filter by status?
PM: Yes — pending, shipped, delivered. Also by date range.
Dev: Validation rules for creating orders?
PM: At least one item per order. Total must be above $10.
     Max 5 orders per customer per day.
Dev: What if a product is out of stock at order time?
PM: Reject the order — no backorders.
Dev: What about cancellations?
PM: Only allowed in pending or confirmed state. Only the customer who placed it can cancel.
Dev: Do we need order history?
PM: Yes, filterable by status and date. Orders older than 3 years go to an archive endpoint.
Dev: Auth?
PM: Everything requires a logged-in user. Admins have a separate role. No guest checkout.
Dev: Rate limiting?
PM: Not decided yet — flag it.
```

---

## Setup & Deployment

### Prerequisites

- A free [GitHub account](https://github.com)
- A free [Cloudflare account](https://cloudflare.com)
- An [Anthropic API key](https://console.anthropic.com)

---

### Part 1 — Deploy the Cloudflare Worker (Proxy)

The Worker holds your Anthropic API key securely as an encrypted secret.

#### 1.1 Create a Cloudflare Worker

1. Go to [workers.cloudflare.com](https://workers.cloudflare.com) and sign in
2. Click **Workers & Pages → Create → Create Worker**
3. Name it `api-analyzer-proxy`
4. Click **Deploy**

#### 1.2 Add the Worker code

1. Open the Worker → click **Edit code**
2. Delete all existing code
3. Paste the contents of `worker/index.js` from this repo
4. Click **Deploy**

#### 1.3 Add your Anthropic API key as a secret

1. Go to your Worker → **Settings → Variables**
2. Under **Environment Variables** → click **Add variable**
3. Set:
   - Variable name: `ANTHROPIC_API_KEY`
   - Value: your key from [console.anthropic.com](https://console.anthropic.com)
   - Click **Encrypt** to make it a secret
4. Click **Save and deploy**

Your Worker URL will be:
```
https://api-analyzer-proxy.YOUR-SUBDOMAIN.workers.dev
```

#### 1.4 Disable Cloudflare Access (important!)

By default, Cloudflare may enable Zero Trust Access on your Worker, which blocks all public requests with a login redirect. To disable it:

1. Go to [one.dash.cloudflare.com](https://one.dash.cloudflare.com) (Zero Trust dashboard)
2. Navigate to **Access → Applications**
3. Find `api-analyzer-proxy - Production`
4. Click the three dots → **Delete**

To verify it's working, open in your browser:
```
https://api-analyzer-proxy.YOUR-SUBDOMAIN.workers.dev/health
```

You should see:
```json
{ "status": "ok", "service": "Meridian Worker", "version": "1.0.0" }
```

---

### Part 2 — Deploy the Frontend on GitHub Pages

#### 2.1 Update the Worker URL in the frontend

Open `index.html` and find this line:
```js
const WORKER_URL = 'https://YOUR-WORKER.workers.dev';
```
Replace it with your actual Worker URL from Part 1.

#### 2.2 Create a GitHub repository

1. Go to [github.com/new](https://github.com/new)
2. Name it `api-analyzer` (or any name)
3. Set visibility to **Public**
4. Click **Create repository**

#### 2.3 Upload the file

**Via GitHub web UI:**
1. Click **Add file → Upload files**
2. Upload `index.html`
3. Commit with message: `Initial commit`

**Via Git CLI:**
```bash
git init
git add index.html
git commit -m "Initial commit"
git branch -M main
git remote add origin https://github.com/YOUR-USERNAME/api-analyzer.git
git push -u origin main
```

#### 2.4 Enable GitHub Pages

1. Go to your repo → **Settings → Pages**
2. Set **Source** to `Deploy from a branch`
3. Set **Branch** to `main` and **Folder** to `/ (root)`
4. Click **Save**

Your site will be live at:
```
https://YOUR-USERNAME.github.io/api-analyzer
```

> It takes about 60 seconds for GitHub Pages to build on first deployment. Watch the progress at **Actions** tab → *pages build and deployment*.

---

### Part 3 — Share with Your Team

Send your team the GitHub Pages URL. No installation, no API key, no login needed on their end.

---

## Project Structure

```
api-analyzer/
├── index.html          # Full frontend — UI, tabs, API calls
└── README.md           # This file

worker/                 # Deploy separately to Cloudflare
└── index.js            # Proxy server — holds API key securely
```

---

## Security Notes

| Concern | How it's handled |
|---------|-----------------|
| API key exposure | Stored as encrypted secret in Cloudflare Worker — never in frontend code |
| CORS | Worker restricts allowed origins — update `ALLOWED_ORIGIN` to your GitHub Pages URL in production |
| Rate limiting | Cloudflare free tier allows 100,000 Worker requests/day |
| Cost control | Each analysis uses ~5,000 tokens (~$0.001 at Haiku pricing) |

To restrict the Worker to only accept requests from your GitHub Pages domain, update line 5 of `worker/index.js`:
```js
// Before (allows all origins):
const ALLOWED_ORIGIN = '*';

// After (restricts to your domain):
const ALLOWED_ORIGIN = 'https://YOUR-USERNAME.github.io';
```

---

## Technology Stack

| Technology | Usage |
|-----------|-------|
| HTML / CSS / Vanilla JS | Frontend UI — no framework, no build step |
| Cloudflare Workers | Serverless proxy — Node.js-compatible runtime |
| Anthropic Claude API | AI analysis — `claude-haiku-4-5` model |
| GitHub Pages | Static site hosting |

---

## Roadmap

Ideas for future versions:

- [ ] Connect to Jira — auto-create tickets from ambiguities
- [ ] Push generated spec directly to Swagger Hub or API gateway
- [ ] Support multiple transcript uploads per session
- [ ] Version comparison — diff specs across meeting iterations
- [ ] Slack / Teams bot integration for in-meeting analysis
- [ ] Auth layer — restrict access to org members only

---

## Contributing

Contributions, ideas, and feedback are welcome. Feel free to open an issue or submit a pull request.

---

## Author

**Rahul Chadha**
GitHub: [@rchadh](https://github.com/rchadh)

---

## License

MIT License — free to use, modify, and distribute.

---

*Built with Claude AI by Anthropic*
