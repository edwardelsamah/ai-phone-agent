[README.md](https://github.com/user-attachments/files/28409896/README.md)
# AI Phone Agent

A self-funded, independently built after-hours voice AI agent that answers calls for businesses, books appointments, takes messages, and answers FAQs — entirely through natural phone conversation. Built with Python, Claude, Twilio, and ElevenLabs.

---

## What it does

When a customer calls a business after hours, instead of hearing a voicemail, they're connected to a conversational AI agent that can:

- **Answer common questions** from a configurable FAQ list
- **Book appointments** by collecting date, time, and reason, then saving to Google Calendar
- **Take messages** with caller name, phone number, and reason
- **Speak naturally in English or French**, detecting the caller's language automatically and switching voices accordingly
- **Send instant notifications** to the business via email and SMS when a message is taken or appointment booked

A web dashboard lets operators manage multiple client businesses, view call transcripts, configure the agent persona and FAQs, and review appointments.

---

## Architecture

```
Caller ──[PSTN]──▶ Twilio Voice API
                        │
          POST /webhook/voice/incoming
                        │
                        ▼
              ┌─────────────────────┐
              │   FastAPI Server    │
              │   (Python/uvicorn)  │
              └────────┬────────────┘
                       │
          ┌────────────┼────────────────┐
          │            │                │
          ▼            ▼                ▼
   Anthropic API   ElevenLabs TTS   SQLite DB
   (Claude)        (streaming MP3)  (calls, appts,
   Generates       Twilio plays       messages)
   structured      audio chunks
   text reply      in real-time
          │
          └──▶ Notifications (email / SMS)
               Google Calendar (appointments)
```

**Call flow, step by step:**

1. Caller dials the Twilio number → Twilio POSTs to `/webhook/voice/incoming`
2. Server greets the caller (ElevenLabs TTS streamed to Twilio via `<Play>`)
3. Twilio's `<Gather>` captures caller speech and POSTs transcript to `/webhook/voice/gather`
4. Server appends the turn, calls the Claude API with full conversation history
5. Claude returns a structured response (`ACTION`, `LANGUAGE`, `TEXT`, appointment fields…)
6. Server queues the reply text, returns TwiML instantly (`<5 ms`), streams ElevenLabs audio on demand
7. Loop repeats until Claude signals `book_appointment`, `take_message`, or `end_call`
8. On completion: DB record updated, Google Calendar event created, email/SMS notification sent

---

## Tech Stack

| Tool | Role | Why |
|---|---|---|
| **FastAPI** | Web framework / webhook server | Async-native, excellent for webhook-heavy workloads, automatic docs |
| **Claude (Anthropic)** | AI brain — conversation + intent detection | Superior instruction-following for structured output; bilingual without fine-tuning |
| **Twilio Voice** | Phone calls, speech-to-text | Industry standard for PSTN bridging; built-in STT via `<Gather>` |
| **ElevenLabs** | Text-to-speech | Natural-sounding voices with sub-second streaming latency — critical for phone UX |
| **SQLite + SQLAlchemy** | Persistence | Zero-config; sufficient for self-hosted / low-volume deployments |
| **pydantic-settings** | Config management | Type-safe env var parsing with `.env` file support |
| **httpx** | Async HTTP client | Streams ElevenLabs MP3 chunks to Twilio in real time |

---

## Prerequisites

You'll need accounts with:

- [Anthropic](https://console.anthropic.com) — Claude API access
- [Twilio](https://www.twilio.com) — a Voice-capable phone number
- [ElevenLabs](https://elevenlabs.io) — at least one voice cloned or selected
- (Optional) [Google Cloud](https://console.cloud.google.com) — service account for Calendar integration
- (Optional) Gmail or [SendGrid](https://sendgrid.com) — for email notifications

For local development you also need:
- [ngrok](https://ngrok.com) or a public IP to expose your local server to Twilio

---

## Installation

```bash
# 1. Clone the repo
git clone https://github.com/your-username/ai-phone-agent.git
cd ai-phone-agent

# 2. Create and activate a virtual environment
python -m venv .venv
# Windows:
.venv\Scripts\activate
# macOS/Linux:
source .venv/bin/activate

# 3. Install dependencies
pip install -r requirements.txt

# 4. Configure environment variables
cp .env.example .env
# Edit .env with your API keys (see Environment Variables section below)
```

---

## Environment Variables

All configuration lives in `.env`. Copy `.env.example` to get started:

| Variable | Required | Description |
|---|---|---|
| `ANTHROPIC_API_KEY` | ✅ | Claude API key from console.anthropic.com |
| `TWILIO_ACCOUNT_SID` | ✅ | From Twilio Console → Account Info |
| `TWILIO_AUTH_TOKEN` | ✅ | From Twilio Console → Account Info |
| `ELEVENLABS_API_KEY` | ✅ | From elevenlabs.io → Profile |
| `ELEVENLABS_VOICE_ID` | ✅ | Primary voice ID (used before language is detected) |
| `ELEVENLABS_VOICE_ID_EN` | ☐ | Optional second voice for English callers |
| `ELEVENLABS_MODEL_ID` | ✅ | Model ID — `eleven_turbo_v2_5` recommended |
| `BASE_URL` | ✅ | Your public server URL (ngrok, IP, or domain) — no trailing slash |
| `SMTP_USER` | ☐ | Gmail address for email notifications |
| `SMTP_PASSWORD` | ☐ | Gmail App Password (not your regular password) |
| `NOTIFICATION_FROM_EMAIL` | ☐ | From-address on notification emails |
| `SENDGRID_API_KEY` | ☐ | Alternative to SMTP for higher-volume email |
| `GOOGLE_CREDENTIALS_JSON` | ☐ | Service account JSON (single line) for Calendar |
| `DATABASE_URL` | ☐ | SQLite path — default `sqlite:///./data/phonagent.db` |
| `SECRET_KEY` | ✅ | Random secret — generate with `python -c "import secrets; print(secrets.token_hex(32))"` |

---

## Running Locally

```bash
# Start the development server (auto-reloads on file changes)
python -m uvicorn app.main:app --reload --port 8000

# In a separate terminal, expose it to Twilio via ngrok
ngrok http 8000
```

Copy the ngrok HTTPS URL (e.g. `https://abc123.ngrok-free.app`) and:

1. Set `BASE_URL=https://abc123.ngrok-free.app` in your `.env`
2. In Twilio Console → Phone Numbers → your number → Voice Configuration:
   - Set webhook to `https://abc123.ngrok-free.app/webhook/voice/incoming` (HTTP POST)

The dashboard is served at `http://localhost:8000` — create your first client there.

---

## Project Structure

```
ai-phone-agent/
├── app/
│   ├── main.py                  # FastAPI app factory, router mounts, logging
│   ├── config.py                # pydantic-settings — all env vars in one place
│   ├── database.py              # SQLAlchemy engine, session, auto-migrations
│   ├── models.py                # ORM models: Client, Call, CallTurn, Appointment, FAQ
│   ├── schemas.py               # Pydantic request/response schemas
│   ├── routers/
│   │   ├── webhook.py           # Core voice loop — /webhook/voice/incoming + /gather
│   │   ├── audio.py             # ElevenLabs streaming endpoint — GET /audio/{id}/stream
│   │   ├── clients.py           # CRUD + stats — /api/clients
│   │   ├── calls.py             # Call log with transcripts — /api/clients/{id}/calls
│   │   ├── appointments.py      # Appointment management
│   │   └── faqs.py              # Per-client FAQ management
│   └── services/
│       ├── claude_client.py     # Prompt builder + Anthropic SDK wrapper
│       ├── conversation.py      # In-memory call state (keyed by Twilio CallSid)
│       ├── elevenlabs_service.py# Async streaming TTS with per-voice queuing
│       ├── twiml_builder.py     # TwiML response factories (Gather, Play, Say, Hangup)
│       ├── calendar_service.py  # Google Calendar API integration
│       └── notification.py      # Email (SMTP/SendGrid) + Twilio SMS notifications
├── frontend/
│   ├── index.html               # Dashboard overview
│   ├── client.html              # Client detail: Config / Call Log / Appointments tabs
│   ├── new-client.html          # New client onboarding form
│   ├── css/dashboard.css        # Design system (CSS variables, shared components)
│   └── js/
│       ├── api.js               # fetch() wrapper for all /api/* endpoints
│       ├── dashboard.js         # Overview page logic
│       └── client.js            # Client detail logic
├── .env.example                 # All required variables — copy to .env
├── requirements.txt
├── render.yaml                  # Render.com deployment config
├── startup_server.bat           # Windows: drop in Startup folder to auto-start
├── start_server.bat             # Windows: production start with file logging
└── install_task.ps1             # Windows: Task Scheduler registration (run as Admin)
```

---

## How a Call Works (Under the Hood)

```
1. Twilio POST /webhook/voice/incoming
   └── Match To-number → Client record in DB
   └── Create Call row (status=active)
   └── init_call() → in-memory CallState (turns, language, caller info)
   └── Return TwiML: <Gather><Play>{greeting_audio}</Play></Gather>

2. Twilio POST /webhook/voice/gather  (loops per turn)
   └── Append caller speech to state.turns
   └── Call Claude API → structured response:
       ACTION: continue | book_appointment | take_message | end_call
       LANGUAGE: en | fr
       TEXT: <spoken reply>
       APPOINTMENT_DATE / TIME / REASON / CALLER_NAME / MESSAGE_REASON
   └── Update language state (switches ElevenLabs voice + Twilio ASR language)
   └── queue_speech(text, voice_id) → returns UUID instantly (no network call)
   └── Return TwiML: <Gather><Play>/audio/{uuid}/stream</Play></Gather>

3. Twilio GET /audio/{uuid}/stream
   └── pop_pending_text(uuid) → retrieve queued (text, voice_id)
   └── Stream POST to ElevenLabs → yield MP3 chunks → StreamingResponse
   └── Twilio plays audio as chunks arrive (first audio in ~300ms)

4. On book_appointment / take_message / end_call
   └── Write to DB, create Google Calendar event, send email + SMS
   └── end_call(call_sid) → remove from in-memory store
   └── Return TwiML: <Play>{farewell}</Play><Hangup/>
```

---

## Bilingual Support (English / French)

The agent automatically detects whether a caller is speaking English or French from their first utterance and responds in that language for the rest of the call:

- **Speech recognition**: Twilio's `<Gather language="fr-CA">` / `"en-US"` attribute is set dynamically based on detected language (defaults to `fr-CA` before detection — safer for bilingual environments)
- **Text-to-speech**: Two ElevenLabs voice IDs can be configured — one primary, one for English — and the server switches between them per-turn
- **AI responses**: Claude is instructed to detect language from the first word and respond entirely in the caller's language, including translating all business information (FAQs, hours, addresses)
- **Per-client default**: Each business can configure whether calls default to English or French before the caller's language is detected

---

## Known Limitations

- **Single-process state**: Active call state is held in memory. If the server restarts mid-call, that call's context is lost. A Redis-backed session store would fix this for production.
- **SQLite**: Works well for self-hosted / low-volume use. Swap `DATABASE_URL` for a Postgres connection string if you need concurrent writes or horizontal scaling.
- **No authentication**: The dashboard has no login. Suitable for private networks; add HTTP Basic Auth or an API gateway before exposing it publicly.
- **Twilio STT accuracy**: Twilio's built-in speech recognition can struggle with accents or background noise. Integrating Deepgram's streaming STT would improve accuracy.
- **Single timezone per client**: Appointment times are stored in UTC and converted per client. Multi-timezone businesses would need extra logic.

## Future Improvements

- [ ] WebSocket-based real-time transcript view in the dashboard
- [ ] Deepgram streaming STT for better accuracy
- [ ] Redis session store for multi-instance deployments
- [ ] Webhook signature verification (Twilio request validation)
- [ ] OAuth login for the dashboard

---

## License

MIT
