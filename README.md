# RFP Case File — SPS

A pre-bid RFP compliance review tool for SPS. Upload one or more documents that make up a
single RFP (PDF/DOCX/TXT — e.g. the main RFP plus its exhibits), optionally describe your
company's capabilities and gaps, pick which departments should review it, and get **one**
Go / Caution / No-Go verdict against SPS's bidding policy — the RFP is always treated as
one case, no matter how many files it's split across. Every deliverable is still traced
back to the exact file, section, and page it came from, so the checklist stays auditable
even when it's pulling from several documents at once.

The frontend has **two analysis engines**, switchable from a toggle in the UI:

| Mode | How it works | Data leaves the browser? |
|---|---|---|
| **AI** | Frontend extracts text locally, then sends it to this repo's backend, which calls **Gemini** (default) or **Claude** to reason about the document and return the analysis. | Yes — document text is sent to the backend and to the AI provider. |
| **Offline** | A hand-coded rule engine runs entirely in the browser (regex/keyword matching and heading detection against the same SPS policy). No network calls at all. | No. |

Both engines return the exact same data shape, so the results UI, exports, and everything
else works identically regardless of which one produced the case file.

## Features

- **Document intake** — PDF, DOCX, or TXT, **multiple files per RFP** (up to 10 files, 10 MB
  each), extracted entirely client-side before any network call is made.
- **One case per RFP, however many files it took** — compliance, evaluation criteria,
  capability fit, score, and the final verdict are all computed once, across every uploaded
  document together, since that's how an RFP package is actually judged. Nothing gets
  split into separate per-file verdicts.
- **Department scope** — restrict a review to any combination of Financial, Legal,
  Technical, and Operations.
- **Capability match** — describe your company's capabilities and gaps in free text; the
  system checks which ones are actually relevant to *this* RFP's requirements (across all
  its documents) and factors relevant gaps into the fit score and verdict.
- **Two-level deliverables tree** — deliverables are grouped under each source document's
  *own* section headings (not a generic template). When more than one file is uploaded,
  section titles are prefixed with their source filename so identically-named sections
  (e.g. "Cost Proposal" in both the main RFP and an exhibit) are never merged. Each item
  shows:
  - **Responsible** (editable) and **Sub-type** (editable) — Narrative, Form,
    Certification, Pricing/Cost, Reference, Resume/Key Personnel, Other
  - **Source file + page** (read-only) — exactly which uploaded file, and which page in
    it, the item was found on; shown without a page number for DOCX/TXT, which have no
    page concept
  - **Status** — starts "Auto"; flips to "Manual" the instant you edit a row, so you can
    always tell what the system inferred vs. what's been human-confirmed
  - Full CRUD: add/remove sections and items, inline title editing, auto-renumbering
- **Compliance checklist + verdict** — per-department checklist rolls up into a 0–100 fit
  score and a stamped Go / Caution / No-Go verdict. Any hard policy violation (e.g. an
  insurance requirement over SPS's cap) forces No-Go regardless of other scores.
- **Export** — a dropdown next to the results lets you export the full case file as either
  **.txt** or a formatted **.pdf** (generated client-side via jsPDF, no backend involved).

## Project structure

```
rfp-analyzer/
├── backend/          Express server, calls Gemini or Anthropic
│   ├── server.js
│   ├── package.json
│   └── .env.example
└── frontend/
    └── index.html    Single-file frontend (no build step)
```

## Setup

### 1. Backend (required for AI mode)

```bash
cd backend
npm install
cp .env.example .env
```

Edit `.env`. By default the backend uses **Gemini**, which has a genuine free tier:

```
AI_PROVIDER=gemini
GEMINI_API_KEY=your-key-here
```

Get a free Gemini key (no credit card) at **https://aistudio.google.com/apikey**.

To use Claude instead, set:

```
AI_PROVIDER=anthropic
ANTHROPIC_API_KEY=sk-ant-...
```

Note: the Anthropic API requires a funded Console account — if you see a
`"credit balance is too low"` error, either add credits at
console.anthropic.com/settings/billing or switch `AI_PROVIDER` back to `gemini`.

Then start the server:

```bash
npm start
```

You should see something like:

```
RFP analyzer backend listening on http://localhost:3001
Using provider: gemini (gemini-2.5-flash)
```

Check it's up:

```bash
curl http://localhost:3001/api/health
```

### 2. Frontend

No build step — it's a single HTML file. Either:

- Open `frontend/index.html` directly in a browser, or
- Serve it so it behaves more like a real deployment: `npx serve frontend`

On load it defaults to **AI mode** and pings the backend automatically; the status line
under the engine toggle tells you if it can't reach it (and which provider/model it found).
If you don't want to run the backend at all, just click **Offline** — the local rule engine
needs nothing else.

If your backend runs somewhere other than `http://localhost:3001`, set it before the page
loads:

```html
<script>window.RFP_API_BASE = 'https://your-backend.example.com';</script>
```
(add this line just above the closing `</head>` tag in `frontend/index.html`)

## Notes

- The rate limiter caps AI analysis at 20 requests / 15 minutes per IP to protect the
  provider quota — adjust in `backend/server.js` if needed.
- Document text sent to `/api/analyze` is capped at 60,000 characters across ALL uploaded
  files combined; if you're routinely submitting large multi-exhibit RFPs in AI mode and
  hitting that cap, raise `MAX_CHARS` in `backend/server.js`.
- Deliverables are extracted only from the submission-requirements portion of each document
  and explicitly stop before any Evaluation Criteria / Scoring section, so the two never
  bleed into each other.
- When multiple files are uploaded, they're combined into a single request with explicit
  `=== FILE: name ===` and `[PAGE n]` delimiters, and the AI is instructed to treat them as
  one RFP package — one verdict, one score, one checklist — while still using those markers
  (never quoted as content) to attribute each deliverable's exact `sourceFile` and
  `sourcePage`. The offline engine does the equivalent using each file's own extracted text
  and real page-offset map, merged into one combined case.
- The AI system prompt encodes the same SPS policy rules as the offline engine (NET30
  payment terms, $5M insurance cap, unaudited financials accepted, etc.), so the two modes
  are judged against the same standard and are meaningful to compare.
- Switching providers is a one-line `.env` change (`AI_PROVIDER`) — no frontend or code
  changes needed, since both providers are prompted to return the same JSON shape.
- If you're on Gemini and see analysis fail with a token-limit/truncation error, check that
  `thinkingConfig.thinkingBudget` is set to `0` in `backend/server.js` — Gemini 2.5 models
  think by default, and thinking tokens are deducted from the same output budget as the
  actual answer, which can silently eat the whole response.
