# Signal — Call Intelligence Dashboard

> A beautiful, self-hosted call grading dashboard for lawn care and home services businesses. Connects to GoHighLevel via webhooks to automatically grade, categorize, and analyze every inbound call.

![Signal Dashboard](https://serviceareaexpert.com/signal/)

## What It Does

Signal turns raw call transcripts from GoHighLevel into actionable intelligence. Every call gets automatically:

- **Graded A–F** based on how well it was handled
- **Categorized** as Lead, Existing Client, Bad Lead, Spam, Tire Kicker, or Wrong Number
- **Scored** across 5 dimensions: Answered, Qualified, Appointment Booked, Follow-Up Promised, Professional Tone
- **Flagged** if it's a Missed Opportunity (real lead that got a D or F grade)

## Live Demo

👉 **[https://serviceareaexpert.com/signal/](https://serviceareaexpert.com/signal/)**

Loaded with 25 realistic sample calls. Click any row to expand the full transcript, score breakdown, and key phrases.

---

## Features

- 📊 **KPI Cards** — Total calls, leads captured, grade distribution, missed opportunities, avg duration
- 🍩 **Grade Donut Chart** — Visual A/B/C/D/F breakdown with warm color coding
- 📈 **14-Day Volume Chart** — Area chart showing call trends
- 📋 **Sortable/Filterable Table** — Filter by grade, category, missed opportunity; search by name or phone
- 🔍 **Expandable Call Rows** — Full transcript, AI summary, score breakdown, key phrases
- ⚡ **Webhook Simulator** — Test button injects realistic calls without needing GHL connected
- 📥 **CSV Export** — One-click export of filtered data
- 💾 **localStorage Persistence** — Data survives page refreshes

---

## Stack

| Layer | Tech |
|-------|------|
| Frontend | Vanilla HTML/CSS/JS (zero dependencies, zero build step) |
| Data ingestion | GoHighLevel → GHL Workflow → Webhook → n8n |
| AI grading | n8n + Claude/GPT (grades each transcript via API) |
| Deployment | Any static file host (SiteGround, Vercel, Netlify, S3) |

---

## How It Works

```
Call ends in GHL
    ↓
Transcript Generated trigger fires
    ↓
GHL Workflow webhooks transcript to n8n
    ↓
n8n sends transcript to Claude/GPT with grading prompt
    ↓
AI returns grade, category, scores, key phrases
    ↓
n8n POSTs structured data to Signal dashboard endpoint
    ↓
Dashboard updates in real time
```

---

## Setup Guide

### 1. Deploy the Dashboard

**Option A — Drop a file anywhere:**
```bash
# Just upload index.html to any web server
scp index.html user@yourserver:/var/www/signal/index.html
```

**Option B — SiteGround (SAE pattern):**
```bash
ssh -i ~/.ssh/your_key -p 18765 user@yourhost \
  "mkdir -p ~/www/yourdomain.com/public_html/signal"
scp -i ~/.ssh/your_key -P 18765 index.html \
  user@yourhost:~/www/yourdomain.com/public_html/signal/index.html
```

### 2. Configure GoHighLevel

1. Go to **Automation → Workflows** in your GHL sub-account
2. Create a new workflow → **Start from Scratch**
3. Add trigger: **Transcript Generated**
4. Enable filters:
   - Call Direction: `Inbound`
   - Call Duration: Greater than `10` seconds (filters out hang-ups)
5. Add action: **Webhook**
   - Method: `POST`
   - URL: `https://your-n8n-instance.com/webhook/call-graded`
   - Body: Include `{{full transcript}}`, `{{contact.id}}`, `{{contact.name}}`, `{{contact.phone}}`, `{{call.duration}}`, `{{call.type}}`

### 3. Set Up n8n Workflow

Import the n8n workflow (coming soon — `n8n/call-grading-workflow.json`).

The workflow:
1. Receives the webhook from GHL
2. Sends transcript to Claude/GPT with the grading prompt below
3. Parses the AI response
4. POSTs structured call data to your dashboard

**AI Grading Prompt:**
```
You are a call quality analyst for a lawn care marketing agency.

Analyze this call transcript and return a JSON object with:
- grade: "A" | "B" | "C" | "D" | "F"
- category: "Lead" | "Existing Client" | "Bad Lead" | "Spam" | "Tire Kicker" | "Wrong Number"
- sentiment: "Positive" | "Neutral" | "Negative"
- score: { answered: bool, qualified: bool, appointmentBooked: bool, followUpPromised: bool, professionalTone: bool }
- keyPhrases: string[] (up to 5 notable phrases from the call)
- summary: string (1-2 sentence summary)
- missedOpportunity: bool (true if category=Lead AND grade=D or F)

Grading rubric:
A = Lead identified, qualified, appointment booked or strong follow-up
B = Lead qualified, follow-up promised, professional handling
C = Adequate — call handled but no clear next step
D = Poor handling — lead likely lost due to agent behavior
F = Call mishandled, rude, or opportunity completely missed

Transcript:
{{transcript}}

Return ONLY valid JSON, no explanation.
```

### 4. Wire Up Live Data

The dashboard accepts POST requests to update its data. Since it's a static HTML file, live data is stored in `localStorage`. For production use with real-time multi-user updates, see [Production Setup](#production-setup) below.

---

## Data Schema

```json
{
  "id": "uuid",
  "timestamp": "2026-03-28T14:32:00.000Z",
  "contactName": "Marcus Webb",
  "contactPhone": "(813) 555-0192",
  "duration": 147,
  "direction": "inbound",
  "callType": "LC Phone",
  "transcript": "Full call transcript text...",
  "summary": "Prospective client expressed interest in weekly lawn service.",
  "category": "Lead",
  "grade": "B",
  "score": {
    "answered": true,
    "qualified": true,
    "appointmentBooked": false,
    "followUpPromised": true,
    "professionalTone": true
  },
  "sentiment": "Positive",
  "keyPhrases": ["ready to book", "asked about pricing"],
  "missedOpportunity": false,
  "locationId": "your-ghl-location-id",
  "clientName": "Your Client Name"
}
```

---

## Production Setup

For a multi-client, real-time production setup:

1. **Add a lightweight backend** (Node/Express or a serverless function) to receive POST data and persist to a database instead of localStorage
2. **Auth layer** — add a simple token check so only your n8n instance can write data
3. **Per-client isolation** — filter by `locationId` to scope each client's view
4. **Deploy on Vercel/Railway** for zero-ops hosting

---

## Customization

### Color Palette
All colors are CSS custom properties at the top of `index.html`:
```css
:root {
  --accent-amber: #d4956a;   /* Primary accent */
  --accent-sage: #7a9e8a;    /* Grade A / Leads */
  --grade-a through --grade-f /* Grade colors */
}
```

### Grade Weights
Modify the AI grading prompt in your n8n workflow to adjust what earns an A vs B.

### Categories
Add custom categories by extending the `CATEGORIES` array and adding matching CSS classes.

---

## Roadmap

- [ ] n8n workflow JSON export
- [ ] Multi-client view (agency dashboard showing all sub-accounts)
- [ ] Email/Slack alert when missed opportunity detected
- [ ] Weekly digest email (top missed opps, grade trend)
- [ ] Light mode
- [ ] Per-agent performance tracking (which team member handled the call)
- [ ] GHL contact auto-tagging based on grade

---

## Built By

[Lawn & Land Marketing](https://lawnandlandmarketing.com) — Digital marketing for lawn care and landscaping businesses.

Signal is built and maintained by the L&L automation team. If you're a lawn care business looking for done-for-you marketing, [get in touch](https://lawnandlandmarketing.com).

---

## License

MIT — use it, fork it, build on it.
