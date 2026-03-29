# CallCanopy — Call Intelligence Platform

> AI-powered call grading and intelligence dashboard for lawn care and home services businesses. Connects to GoHighLevel via webhooks — every call automatically graded, categorized, and analyzed. No Deepgram, no OAuth complexity. Just GHL → n8n → Claude → dashboard.

**Live Dashboard:** [https://canopy.groundcontrol.agency](https://canopy.groundcontrol.agency)  
**GitHub:** [https://github.com/LawnAndLandMarketing/callcanopy](https://github.com/LawnAndLandMarketing/callcanopy)

---

## What It Does

Every call that comes through GoHighLevel gets automatically:

- **Graded A–F** — how well the call was handled by the rep
- **Categorized** — Lead, Existing Client, Bad Lead, Spam, Tire Kicker, Wrong Number, or Missed Call
- **Scored** across 5 dimensions: Answered, Qualified, Appointment Booked, Follow-Up Promised, Professional Tone
- **Deep intelligence extracted** — services mentioned, objections raised (+ how handled), questions asked, competitor mentions, compliments given
- **Flagged** — Missed Opportunity (real lead that got a D or F)

Missed calls are tracked separately via a second GHL workflow — no transcript needed, just logs that the call went unanswered.

---

## Architecture

```
Call ends in GHL (or goes unanswered)
    ↓
Transcript Generated trigger  ──OR──  Missed Call trigger
    ↓                                        ↓
Webhook → n8n                         Webhook → n8n
    ↓
Claude Haiku grades transcript
    ↓
Structured call record built
    ↓
Dashboard updated (canopy.groundcontrol.agency)
```

**No Deepgram needed.** GHL now provides call transcripts natively via the Transcript Generated trigger. This replaces the previous architecture that required Deepgram nova-2 for transcription.

---

## Stack

| Layer | Tech |
|-------|------|
| Dashboard | Vanilla HTML/CSS/JS — zero dependencies, zero build step |
| Automation | n8n (lawnandlandmarketing.app.n8n.cloud) |
| AI Grading | Anthropic Claude Haiku (claude-haiku-3-20240307) |
| Hosting | Vercel via lawnlab.dev (canopy.groundcontrol.agency) |
| CRM Source | GoHighLevel (Service Area Expert sub-accounts) |

---

## n8n Workflows

Two active workflows in n8n:

### 1. CallCanopy — Call Grading (Transcript Generated)
**Webhook:** `https://lawnandlandmarketing.app.n8n.cloud/webhook/callcanopy-call-graded`

Flow:
1. Receive GHL webhook
2. Immediately respond 200 to GHL (prevents timeout)
3. Extract fields from payload
4. Send transcript to Claude Haiku with grading prompt
5. Build structured call record
6. Store to data layer

### 2. CallCanopy — Missed Call Logger
**Webhook:** `https://lawnandlandmarketing.app.n8n.cloud/webhook/callcanopy-missed-call`

Flow:
1. Receive GHL missed call webhook
2. Log as `missedCall: true`, grade `-`, category `Missed Call`
3. Store to data layer

---

## GHL Setup

### Workflow 1 — Transcript Generated

1. **Automation → Workflows → Create New**
2. Trigger: **Transcript Generated**
3. Filter: Call Duration > 10 seconds, Direction = Inbound
4. Action: **Webhook → POST** to `callcanopy-call-graded` URL
5. Custom data fields:

| Key | Value |
|-----|-------|
| `contactName` | `{{contact.name}}` |
| `contactPhone` | `{{contact.phone}}` |
| `contactId` | `{{contact.id}}` |
| `transcript` | `{{transcript_generated.call_transcript}}` |

Note: `locationId` is hardcoded in n8n per sub-account. `{{location.id}}` is not available as a GHL merge field.

### Workflow 2 — Missed Call

1. Trigger: **Call Status → Missed**
2. Action: **Webhook → POST** to `callcanopy-missed-call` URL
3. Custom data fields:

| Key | Value |
|-----|-------|
| `contactName` | `{{contact.name}}` |
| `contactPhone` | `{{contact.phone}}` |
| `contactId` | `{{contact.id}}` |

---

## AI Grading Prompt

Used in n8n via Claude Haiku API:

```
You are a call quality analyst for a lawn care marketing agency. Analyze this transcript 
and return ONLY valid JSON with these exact keys:

- grade: "A" | "B" | "C" | "D" | "F"
- category: "Lead" | "Existing Client" | "Bad Lead" | "Spam" | "Tire Kicker" | "Wrong Number"
- sentiment: "Positive" | "Neutral" | "Negative"
- missedOpportunity: bool (true ONLY if category=Lead AND grade=D or F)
- summary: string (1-2 sentences)
- keyPhrases: string[]
- score: { answered, qualified, appointmentBooked, followUpPromised, professionalTone } (all bool)
- services: string[]
- objections: [{ content, category (price/timing/trust/other), response }]
- questions: string[]
- competitors: string[]
- compliments: string[]

Grading rubric:
A = Lead identified, qualified, appointment booked or strong follow-up
B = Lead qualified, follow-up promised, professional handling
C = Adequate — call handled but no clear next step
D = Poor handling — lead likely lost due to agent behavior
F = Call mishandled, rude, or opportunity completely missed
```

---

## Data Schema

```json
{
  "id": "string",
  "timestamp": "ISO8601",
  "contactName": "string",
  "contactPhone": "string",
  "contactId": "string",
  "locationId": "string",
  "clientName": "string",
  "duration": 0,
  "direction": "inbound",
  "callType": "LC Phone | Voice AI | IVR",
  "transcript": "string",
  "summary": "string",
  "grade": "A | B | C | D | F | -",
  "category": "Lead | Existing Client | Bad Lead | Spam | Tire Kicker | Wrong Number | Missed Call",
  "sentiment": "Positive | Neutral | Negative",
  "missedOpportunity": false,
  "missedCall": false,
  "keyPhrases": [],
  "score": {
    "answered": true,
    "qualified": true,
    "appointmentBooked": false,
    "followUpPromised": true,
    "professionalTone": true
  },
  "services": [],
  "objections": [{ "content": "string", "category": "string", "response": "string" }],
  "questions": [],
  "competitors": [],
  "compliments": []
}
```

---

## Dashboard Features

- **KPI Cards** — Total calls, leads captured, grade distribution, missed opportunities, avg duration
- **Grade Donut Chart** — A/B/C/D/F breakdown with warm color palette
- **Category Bar Chart** — Call type distribution
- **14-Day Volume Area Chart** — Call trends over time
- **Sortable/Filterable Table** — Filter by grade, category, missed opportunity; search by name/phone
- **Expandable Call Rows** — Full transcript, AI summary, score breakdown, key phrases, services, objections
- **Missed Opportunity Flags** — Glowing red dot on leads with D/F grades
- **CSV Export** — One-click filtered data export
- **Webhook Simulator** — Test button injects realistic calls without GHL connected

---

## Roadmap

### ✅ Phase 1 — Foundation + Multi-Client Agency View (Complete — March 2026)
- Full call grading pipeline: GHL → n8n → Claude Haiku → Supabase → Dashboard
- Missed call tracking via separate GHL workflow
- 9 intelligence categories per call (grade, sentiment, services, objections, questions, competitors, compliments, key phrases, score breakdown)
- **Agency View** — card grid showing all clients with call summary stats (total calls, leads, missed, top grade)
- **Client View** — full dashboard filtered to a single sub-account
- View toggle: Agency ↔ Client in header
- Client selector dropdown (populated from Supabase `clients` table)
- Drill-down: click an agency card → jumps to client view
- Supabase backend (real-time, multi-client, auto-refreshes every 30s)
- Live at canopy.groundcontrol.agency

### 📋 Phase 2 — Client Onboarding Flow
- Per-client GHL workflow setup (locationId hardcoded per sub-account)
- `clients` table in Supabase populated for all active L&L clients
- Add/remove clients via simple Supabase insert
- Unique shareable URL per client: `canopy.groundcontrol.agency?client=slug`

### 📋 Phase 3 — Agency Intelligence
- Cross-client objection trending (what objections are all clients hearing?)
- Agency-wide lead conversion benchmarks
- Client health scores (grading trend over time)
- Weekly digest emails per client (top missed opps, grade distribution)

### 📋 Phase 4 — Client Portal
- Supabase Auth login (clients see only their own data)
- White-labeled per-client subdomain option
- Client notification emails for missed opportunities
- Embeddable widget for client sites

### 📋 Phase 4 — Client Portal
- Client-facing login (clients see only their own data)
- White-labeled reports
- Client notification emails for missed opportunities

### 📋 Phase 5 — Advanced Intelligence
- Per-rep performance scorecards
- Lost deal analysis
- Seasonal trend tracking
- Service area expansion signals
- Pricing intelligence from call data
- Training recommendations

---

## Previous Architecture (Deprecated)

The original CallCanopy (repo: `call-intelligence`) was built as a full Next.js 16 + Supabase + Vercel app with:
- GHL OAuth flow for sub-account connection
- Deepgram nova-2 for call transcription
- Full database schema (clients, communications, extracted_items tables)
- GHL Marketplace app submission (Client ID: `69bb00239b7d03d5bd623b52-mmwgaasu`)

This was simplified after GHL released native transcript generation via the Transcript Generated trigger, making Deepgram and OAuth unnecessary for our use case. The new architecture is faster, cheaper, and easier to maintain.

The Supabase schema and database design from the original build will be used in Phase 3 when we add multi-client persistence.

---

## Built By

[Lawn & Land Marketing](https://lawnandlandmarketing.com) — Digital marketing for lawn care and landscaping businesses. Largo, Florida.

Internal AI assistant: Roshi 🐢
