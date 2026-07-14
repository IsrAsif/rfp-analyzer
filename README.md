# RFP Case File — SPS

A pre-bid RFP compliance review tool for SPS. Upload one or more RFP-related documents
(PDF/DOCX/TXT — e.g. the main RFP plus its exhibits), optionally describe your company's
capabilities and gaps, pick which departments should review it, and get a Go / Caution /
No-Go verdict against SPS's bidding policy for **each file, analyzed independently** —
never combined or renamed, each case keeping the exact filename it was uploaded with.

The frontend has **two analysis engines**, switchable from a toggle in the UI:

| Mode | How it works | Data leaves the browser? |
|---|---|---|
| **AI** | Frontend extracts text locally, then sends it to this repo's backend, which calls **Gemini** (default) or **Claude** to reason about the document and return the analysis. | Yes — document text is sent to the backend and to the AI provider. |
| **Offline** | A hand-coded rule engine runs entirely in the browser (regex/keyword matching and heading detection against the same SPS policy). No network calls at all. | No. |

Both engines return the exact same data shape, so the results UI, exports, and everything
else works identically regardless of which one produced the case file.

## Features

- **Document intake** — PDF, DOCX, or TXT, **multiple files at once** (up to 10 files, 10 MB
  each), extracted entirely client-side before any network call is made.
- **Independent analysis per file** — each uploaded document gets its own complete,
  separate analysis (deliverables, compliance, capability fit, score, verdict). Files are
  never combined or renamed; when more than one file is uploaded, a **case selector** (pill
  row) lets you switch between each file's results, and every case is labeled with its
  exact original filename.
- **Department scope** — restrict a review to any combination of Financial, Legal,
  Technical, and Operations (applied to every file's analysis).
- **Capability match** — describe your company's capabilities and gaps in free text; the
  system checks which ones are actually relevant to *this* RFP's requirements and factors
  relevant gaps into the fit score and verdict.
- **Two-level deliverables tree** — deliverables are grouped under the document's *own*
  section headings (not a generic template). Each item shows:
  - **Responsible** (editable) and **Sub-type** (editable) — Narrative, Form,
    Certification, Pricing/Cost, Reference, Resume/Key Personnel, Other
  - **Source file + page** (read-only) — the exact uploaded file, and the page within it,
    the item was found on; shown without a page number for DOCX/TXT, which have no page
    concept. The server/engine stamps this itself rather than trusting inferred text, so
    it always matches the file you actually uploaded.
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
- Document text sent to `/api/analyze` is capped at 60,000 characters; longer documents are
  truncated with a note appended.
- Deliverables are extracted only from the submission-requirements portion of each document
  and explicitly stop before any Evaluation Criteria / Scoring section, so the two never
  bleed into each other.
- When multiple files are uploaded, each is sent to the AI engine as its own separate
  request — never combined into one prompt — so one document's content, structure, or
  section headings can never leak into another's results. The 60,000-character truncation
  cap therefore applies per file, not across the whole batch.
- The AI system prompt encodes the same SPS policy rules as the offline engine (NET30
  payment terms, $5M insurance cap, unaudited financials accepted, etc.), so the two modes
  are judged against the same standard and are meaningful to compare.
- Switching providers is a one-line `.env` change (`AI_PROVIDER`) — no frontend or code
  changes needed, since both providers are prompted to return the same JSON shape.
- If you're on Gemini and see analysis fail with a token-limit/truncation error, check that
  `thinkingConfig.thinkingBudget` is set to `0` in `backend/server.js` — Gemini 2.5 models
  think by default, and thinking tokens are deducted from the same output budget as the
  actual answer, which can silently eat the whole response.
