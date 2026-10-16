# RailSaathi 🚆

**A multilingual, voice-first Indian train-booking assistant.**

Say it out loud — *"Hyderabad to Vijayawada tomorrow evening, two adults, sleeper,
under ₹500 each"* — and RailSaathi understands the request, asks for anything
that's missing, finds and compares trains, explains the options aloud in your
language, and walks you to the point where you confirm.

> **Status:** working end-to-end demo. Voice input/output and multilingual
> replies run on Sarvam AI. Train data comes from a **clearly labelled,
> deterministic SIMULATED adapter** until you plug in an authorised railway
> provider. RailSaathi never invents train numbers, fares or PNRs.

---

## Contents

- [What it does](#what-it-does)
- [Architecture](#architecture)
- [Project layout](#project-layout)
- [Quick start (local)](#quick-start-local)
- [Environment variables](#environment-variables)
- [Running the tests](#running-the-tests)
- [Sarvam AI integration](#sarvam-ai-integration)
- [Railway data: demo vs live](#railway-data-demo-vs-live)
- [Deploying and getting a public URL](#deploying-and-getting-a-public-url)
- [API reference](#api-reference)
- [Safety and privacy](#safety-and-privacy)
- [Limitations & what needs authorisation](#limitations--what-needs-authorisation)
- [Troubleshooting](#troubleshooting)

---

## What it does

**Voice-first conversation.** A large, accessible microphone button captures one
utterance at a time (push-to-talk style), sends it to Sarvam's Speech-to-Text
API, and speaks the reply back with Sarvam's Text-to-Speech. A full transcript is
shown so you can correct anything the recogniser got wrong, and typed input is
always available as a fallback.

**Journey understanding.** The assistant extracts origin, destination, date,
preferred departure window, number of passengers, travel class and budget. It
resolves relative dates ("tomorrow", "next Monday", "kal") against the real
current date in `Asia/Kolkata`, and **asks instead of guessing** when a date or a
city is ambiguous.

**Real train search, honestly labelled.** A modular provider interface fetches
schedules, durations, classes and fares. When no authorised provider is
configured, a deterministic demo adapter returns realistic but explicitly
**simulated** data — every card is tagged `SIMULATED`, every train number comes
from a reserved `9xxxx` range, and the app shows a `DEMO DATA` badge.

**Safe booking workflow.** Search → clarify → compare → select → passengers →
review → **explicit confirmation** → booking hand-off. RailSaathi refuses card
numbers, OTPs and railway passwords, and never shows a PNR it didn't receive
from an authorised provider.

**Accessibility.** Large touch targets, readable typography, keyboard-accessible
controls, spoken explanations, a visible transcript, a language switcher, clear
loading states and recoverable errors, and a `prefers-reduced-motion` mode.

---

## Architecture

RailSaathi is a genuine multi-agent workflow over a single shared, typed
`JourneyState`. Agents never invent facts and never mutate state on their own —
the orchestrator applies their proposals and owns every stage transition.

```
                    ┌──────────────────────────────────────────┐
  voice ──STT──►    │              Orchestrator                │
  text  ─────────►  │  (owns JourneyState + workflow stages)   │
                    └───────┬──────────────────────────────────┘
                            │
   ┌────────────────────────┼────────────────────────────────────────────┐
   ▼            ▼           ▼            ▼            ▼          ▼        ▼
Intent   Station/Date   Train Search  Recommend-  Booking   Workflow  Response
Agent    Validation     Agent         ation       Validation  &        Agent
(LLM +   Agent         (provider     Agent       Agent       Recovery  (LLM/
rules)   (rules)        interface)   (rules)     (rules)     (rules)   translate)
                                                              │
                                                    ┌─────────┴──────────┐
                                                    ▼                    ▼
                                            DemoRailwayProvider   HttpRailwayProvider
                                            (SIMULATED, default)  (licensed, via env)
```

| Agent | Responsibility | Deterministic? |
| --- | --- | --- |
| **Intent** | Turns one utterance into structured slot updates | Rules fallback, LLM when available |
| **Station & Date Validation** | Resolves stations against a verified table, interprets dates, rejects past dates, reports missing fields | Yes |
| **Train Search** | The only component that talks to a railway provider; owns the timeout | Yes |
| **Recommendation** | Ranks verified options by price, duration, departure window, class availability and preferences | Yes |
| **Booking Validation** | Checks the journey and passenger data, blocks sensitive data | Yes |
| **Workflow & Recovery** | State machine, duplicate-submission guard, alternative suggestions | Yes |
| **Response** | Builds the spoken reply; the LLM/translate layer only *phrases* facts | Facts: yes |

The LLM is used for language understanding and phrasing only. Everything that
reaches the user is either extracted from their own words or returned by the
railway provider.

---

## Project layout

```
railsaathi/
├── backend/
│   ├── app/
│   │   ├── main.py              FastAPI app, routes, health check, static hosting
│   │   ├── config.py            env-driven settings
│   │   ├── models.py            typed domain models (JourneyState, TrainOption, …)
│   │   ├── orchestrator.py      the workflow engine that drives all agents
│   │   ├── sarvam_client.py     async Sarvam API client (STT/TTS/Chat/Translate)
│   │   ├── stations.py          verified station table + resolver
│   │   ├── dateparse.py         deterministic relative-date interpretation
│   │   ├── lang.py              supported languages + script detection
│   │   ├── agents/              intent, validation, search, recommendation,
│   │   │                        booking, workflow, response
│   │   ├── railway/             provider interface, demo adapter, http adapter
│   │   └── data/stations.json   100 stations with codes, cities and aliases
│   ├── tests/                   43 pytest tests
│   ├── requirements.txt
│   └── conftest.py
├── frontend/
│   ├── src/
│   │   ├── App.tsx              voice + text UI, workflow-aware
│   │   ├── api.ts               typed API client
│   │   ├── hooks/useRecorder.ts microphone capture (MediaRecorder)
│   │   ├── components/          Header, JourneyProgress, MicButton, TrainCard,
│   │   │                        Conversation, ChoiceChips, ReviewPanel
│   │   └── styles.css           railway-inspired design system
│   ├── scripts/build.mjs        esbuild production bundler
│   └── package.json
├── Dockerfile                   single-image deploy (frontend + backend)
├── .env.example
├── .gitignore
├── README.md
└── DEMO_SCRIPT.md
```

---

## Quick start (local)

Requires **Python 3.12+** and **Node 20+**.

### 1. Backend

```bash
cd backend
python -m venv .venv && source .venv/bin/activate      # Windows: .venv\Scripts\activate
pip install -r requirements.txt
cp ../.env.example .env        # then edit .env and add your SARVAM_API_KEY
uvicorn app.main:app --reload --port 8000
```

Check it: <http://localhost:8000/api/health> → `{"status":"ok", …}`

### 2. Frontend (development)

```bash
cd frontend
npm install
npm run dev          # http://localhost:5173  (proxies /api to :8000)
```

### 3. Or build the frontend and let the backend serve everything

```bash
cd frontend && npm run build          # writes frontend/dist
cd ../backend && uvicorn app.main:app --port 8000
# open http://localhost:8000
```

> `npm run build` uses a small esbuild bundler (`scripts/build.mjs`) because it
> is fast and dependency-light. `npm run build:vite` runs the equivalent Vite
> production build — both produce the same `dist/`.

---

## Environment variables

Copy `.env.example` to `.env`. **The `.env` file is git-ignored — never commit it.**

| Variable | Required | Default | Purpose |
| --- | --- | --- | --- |
| `SARVAM_API_KEY` | for voice | *(empty)* | Sarvam API subscription key. Server-side only. |
| `SARVAM_BASE_URL` | no | `https://api.sarvam.ai` | Sarvam API base URL. |
| `SARVAM_STT_MODEL` | no | `saaras:v3` | Speech-to-text model (`saaras:v3` or `saaras:v4`). |
| `SARVAM_TTS_MODEL` | no | `bulbul:v3` | Text-to-speech model. |
| `SARVAM_TTS_SPEAKER` | no | `anushka` | TTS voice. |
| `SARVAM_CHAT_MODEL` | no | `sarvam-105b` | Chat model for intent extraction. |
| `SARVAM_CHAT_MODEL_FAST` | no | `sarvam-105b-conversations` | Low-latency chat model. |
| `REQUEST_TIMEOUT` | no | `30` | Per-request timeout (seconds). |
| `RAILWAY_PROVIDER` | no | `demo` | `demo` or `http`. |
| `RAILWAY_API_BASE` | for live | *(empty)* | Base URL of your authorised railway provider. |
| `RAILWAY_API_KEY` | for live | *(empty)* | Key for that provider. |
| `APP_ENV` | no | `development` | Free-form environment label. |
| `CORS_ORIGINS` | no | `http://localhost:5173,http://127.0.0.1:5173` | Allowed browser origins. |

**Without `SARVAM_API_KEY`** the app still runs end to end: the UI shows a banner,
voice endpoints return `503` with a clear message, and text input drives the full
workflow using deterministic extraction. That is the mode used by the automated
tests.

---

## Running the tests

```bash
cd backend
python -m pytest -q
```

43 tests cover date interpretation (including ambiguous `kal`), station
resolution and city-level ambiguity, ranking and budget filtering, workflow
transitions and the duplicate-booking guard, the demo provider's determinism and
labelling, and the API surface (health, config, chat, station search, voice
auth guard).

---

## Sarvam AI integration

Implemented in `backend/app/sarvam_client.py`, following
<https://docs.sarvam.ai>. All calls are server-side; the key never reaches the
browser, source maps or logs.

| Capability | Endpoint | Model (default) | Notes |
| --- | --- | --- | --- |
| Speech-to-Text | `POST /speech-to-text` | `saaras:v3` | multipart; `mode=transcribe`; `language_code` in BCP-47 or `unknown` for auto-detect; ≤30 s per REST request |
| Text-to-Speech | `POST /text-to-speech` | `bulbul:v3` | JSON; returns base64 WAV (`audios[]`); ≤2500 chars |
| Chat Completion | `POST /v1/chat/completions` | `sarvam-105b` | OpenAI-compatible; JSON mode used for intent extraction |
| Translate | `POST /translate` | — | used to localise replies when the user's language isn't English |

Authentication uses the `api-subscription-key` header. Transient failures
(`429`, `5xx`, network) are retried with backoff. Sarvam returns `403` (not
`401`) for auth problems — the client handles both.

If you change model IDs or parameters, verify them against the docs at
implementation time; they are all environment-overridable so you never have to
edit code.

---

## Railway data: demo vs live

No free, public, *authorised* Indian Railways booking API exists, and RailSaathi
does not scrape or bypass IRCTC. The railway layer is therefore modular:

**Demo (default).** `DemoRailwayProvider` generates deterministic options seeded
by `(origin, destination, date)` — so the same route always yields the same
trains and the tests are stable. Every record carries `source="DEMO"` and
`simulated=true`, train numbers come from a reserved `9xxxx` range, names end in
`(Simulated)`, and the UI shows a `DEMO DATA` badge. **Fares and availability are
modelled, not live.**

**Live.** Set `RAILWAY_PROVIDER=http` plus `RAILWAY_API_BASE` and
`RAILWAY_API_KEY`. `HttpRailwayProvider` calls `GET {base}/trains?from=&to=&date=&class=&passengers=`
with a bearer token and expects:

```json
{
  "trains": [
    {
      "train_no": "12704",
      "train_name": "Falaknuma Express",
      "from": "SC", "to": "BZA",
      "departure": "15:55", "arrival": "21:00",
      "duration_minutes": 305, "distance_km": 276,
      "runs_on": "Daily",
      "classes": [
        { "code": "SL", "fare": 320, "availability": "AVAILABLE-42", "status": "AVAILABLE" }
      ]
    }
  ]
}
```

Adapt `_map_option()` in `backend/app/railway/http_provider.py` for any other
provider's shape — nothing else in the app needs to change.

---

## Deploying and getting a public URL

### Option A — one service (recommended, simplest)

The backend serves the built frontend, so you deploy a single URL.

**With Docker:**

```bash
docker build -t railsaathi .
docker run -p 8000:8000 -e SARVAM_API_KEY=your_key railsaathi
```

**On Render / Railway / Fly.io:**

1. Push this repo to GitHub.
2. Create a **Web Service** from the repo; pick the Dockerfile (or set build
   command `cd frontend && npm install && npm run build` and start command
   `cd backend && uvicorn app.main:app --host 0.0.0.0 --port $PORT`).
3. Add the environment variables from `.env.example` — at minimum
   `SARVAM_API_KEY`. Do **not** set `RAILWAY_PROVIDER=http` unless you have a
   provider.
4. Deploy. The platform gives you a public URL like
   `https://railsaathi.onrender.com`. Open it — that's your demo link.

> Note: browsers require **HTTPS** (or `localhost`) for microphone access, so a
> plain-HTTP IP will not offer voice. Use the platform's HTTPS URL.

### Option B — split frontend and backend

- Deploy `frontend/` to **Vercel/Netlify** with build `npm run build`, output
  `dist`, and `VITE_API_BASE=https://your-backend.example.com`.
- Deploy `backend/` as above and add your frontend origin to `CORS_ORIGINS`.

---

## API reference

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` | `/api/health` | Service + provider health, model IDs, demo/live flag |
| `GET` | `/api/config` | Supported languages, voice availability, disclaimer |
| `GET` | `/api/stations?q=` | Station lookup against the verified table |
| `POST` | `/api/session?language=` | Start a session (returns the greeting) |
| `POST` | `/api/session/{id}/reset` | Clear a journey |
| `POST` | `/api/chat` | Text turn → `TurnResult` |
| `POST` | `/api/voice/transcribe` | Audio → `{transcript, language_code}` |
| `POST` | `/api/voice/speak` | Text → `audio/wav` |
| `POST` | `/api/voice/turn` | Audio → transcript + `TurnResult` + optional spoken WAV |

Interactive docs are at `/docs` when the server is running.

---

## Safety and privacy

- **No sensitive data.** RailSaathi actively detects and refuses card numbers,
  OTPs and railway passwords (`agents/booking.py`). It never collects or stores
  payment details.
- **Explicit confirmation.** Booking is only ever attempted after the user
  reviews the journey and confirms, and a duplicate-submission guard prevents
  double bookings.
- **No fake confirmations.** Because no authorised booking provider is
  configured, the final step is an honest **hand-off** to IRCTC — it is
  described as a hand-off, never as a completed booking. A PNR is only ever
  shown if an authorised provider returned it.
- **Server-side keys.** `SARVAM_API_KEY` lives only in the backend process.
- **No CAPTCHA bypass.** RailSaathi does not circumvent any railway protection.

---

## Limitations & what needs authorisation

**Working today:** multilingual voice in/out, journey understanding with
clarification, deterministic ranking, review and explicit confirmation, the
hand-off, accessibility features, and 43 passing tests.

**Not yet wired to real services (and honest about it):**

1. **Live railway data** — needs an authorised provider. Configure
   `RAILWAY_PROVIDER=http` + `RAILWAY_API_BASE` + `RAILWAY_API_KEY`. Until then
   all train data is labelled `SIMULATED`.
2. **Actual booking / PNR** — Indian Railways bookings require an authorised
   channel (IRCTC or a licensed agent). RailSaathi performs a hand-off. No PNR is
   fabricated.
3. **Real-time streaming voice** — this MVP is reliable **turn-by-turn**
   push-to-talk, not continuous duplex conversation. Sarvam offers realtime
   streaming STT/TTS (WebSocket) that could upgrade this; it was not implemented
   here because it needs a persistent socket and careful turn-taking, and the
   turn-by-turn path is more robust for a live demo.
4. **Live availability** — depends on (1).
5. **Persistence** — sessions are in-memory. Add Redis/Postgres for multi-instance
   deployments.

---

## Troubleshooting

| Symptom | Fix |
| --- | --- |
| "Voice input needs SARVAM_API_KEY" | Add the key to `backend/.env` and restart the backend. |
| Mic button says "Voice unavailable" | The browser blocked the mic, or the page isn't on HTTPS/localhost. |
| `vite build` fails with a WebAssembly error | Some restricted containers can't instantiate the bundler's WASM. Use `npm run build` (esbuild path) instead. |
| No trains returned | The route may have no direct service; RailSaathi suggests nearby stations and the next day. |
| CORS error in the browser console | Add your frontend origin to `CORS_ORIGINS`. |

---

Built for a national-level AI hackathon. Sarvam AI handles the speech and
language; the railway facts are the provider's, and RailSaathi says so out loud.
