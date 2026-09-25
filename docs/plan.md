# Jarvis: design plan

_Status: draft for discussion. Nothing is built yet._

## Decisions so far

| Question | Decision |
|---|---|
| How you talk to it | Phone chat **and** voice. Eventually its own phone number. |
| Brain | Any model, through **OpenRouter** |
| Where it runs | Open. The recommendation below is a small cloud server. |

## 1. The big picture

Jarvis is **one core program** with several **front doors**. Every front door leads to the same brain, memory and tools. That way, Jarvis on a phone call knows what you told it on Telegram that morning.

```
 FRONT DOORS                           CORE (one program, always running)
 ───────────                           ─────────────────────────────────
 Telegram chat  ──┐
 SMS to its number ├──► message router ─► AGENT ─► OpenRouter ─► any model
 Phone call ──► voice pipeline ─┘            │
   (speech ↔ text)                          ├─► MEMORY  (notes + database)
                                            ├─► TOOLS   (Gmail, Calendar, web…)
                                            └─► SCHEDULER (morning brief, checks)
```

## 2. Each part in detail

### Front door 1: Telegram (build first)
- It's free, takes about 10 minutes to set up (you create a bot by messaging Telegram's `@BotFather`), and supports voice notes, photos and files.
- **Security:** Jarvis only answers **your** Telegram user ID and ignores everyone else.
- Why not WhatsApp first: WhatsApp's business API needs Meta to verify a business and charges per message. iMessage needs a Mac running all the time. Both are possible later.

### Front door 2: its own phone number (Twilio)
- Rent a number from **Twilio** (a phone-service provider for developers). Then:
  - **Text it:** SMS arrives at Jarvis, same as Telegram.
  - **Call it:** the call audio streams to the voice pipeline below.
  - **It calls or texts you:** for example "Your 3pm got moved, want me to reschedule the gym?"
- Rough cost: a number is about **US$1–3/month**, plus a few cents per minute of calls or per text. Some countries (Australia, for example) need an ID/address check before you can rent a local number.
- **Security:** anyone can dial a phone number, and caller ID can be faked. So:
  - Jarvis hangs up on (or just takes a message from) any number that isn't yours.
  - Even when the call is "from you", sensitive actions (sending email, deleting, paying) need a spoken PIN or a confirmation tap on Telegram.

### Front door 3: voice (the pipeline)
A phone call works like this, in a loop that aims to reply in about 1 second:

1. **Speech-to-text** (for example Deepgram): your voice becomes text, streamed live.
2. **Brain** (via OpenRouter): decides what to say or do.
3. **Text-to-speech** (for example Cartesia or ElevenLabs): the reply is spoken. This is where you pick Jarvis's voice.

**Pipecat** (an open-source toolkit) wires this together. It has official integrations for **both OpenRouter and Twilio**, and a ready-made phone-bot starter project, so we don't build the tricky audio parts ourselves.

**Important trade-off:** in voice, speed matters more than cleverness. The plan is to use:
- a **fast model for conversation** (the voice brain), and
- a **smarter model for real work**. For longer jobs, voice Jarvis says "on it, I'll text you" and hands the task to the main agent in the background.

OpenRouter makes this easy, because switching models just means changing a name.

Later: voice in an app or on your laptop without a phone call (same pipeline, different "transport"), and a wake word ("Hey Jarvis").

### The brain: OpenRouter
- OpenRouter is **one account and one API key that reaches hundreds of models** (Claude, GPT, Gemini, Llama, and others). You can switch models per task and compare cost and quality.
- **Things to know:**
  - Not every model is good at **tool use** (acting on your apps). For the main agent, use a strong one (for example a Claude or GPT model). Cheap models are fine for simple jobs like summarising.
  - OpenRouter adds a small delay and fee on top of the model provider's price. That's worth it for the flexibility.
  - Set a **monthly credit limit** in OpenRouter from day one.
- **Framework change from the earlier research:** the Claude Agent SDK is built around Claude models. Since you want any model, the better fit is **Pydantic AI**, a Python agent framework with built-in OpenRouter support. Python also matches Pipecat, so the whole project is in one language.

### Memory
Two layers:
1. **Markdown notes you can read and edit**: `about-me.md` (preferences, people, routines), `todo.md`, and a daily journal it writes. The agent reads these every time. If it gets something wrong about you, you open the file and fix it.
2. **A database** for conversation history and search across everything ("what did I say about the Brisbane trip last month?"). **SQLite** is a single file, which is enough to start. **Supabase/Postgres**, which you already have, is the upgrade path.

### Tools ("hands")
- These use **MCP**, the standard plug format covered in the research.
- **Important:** the connectors you use inside Claude (Gmail, Calendar, Strava, etc.) belong to the Claude app. **Your own Jarvis can't borrow them.** It needs its own connections, either:
  - **Composio** (which you already have an account with), which handles the logins for many apps, or
  - the official Google or other MCP servers, set up with your own login.
- Start **read-only** (read email, read calendar, web search). Then add write actions one by one, each needing your confirmation at first.

### Scheduler ("heartbeat")
- Timed jobs, for example:
  - a 7am brief by Telegram or voice call,
  - an hourly check that says nothing unless something is urgent,
  - reminders.
- This is what makes it feel like Jarvis rather than a chatbot.

## 3. Where it runs: recommendation

**A small cloud server** (for example Hetzner, DigitalOcean or Fly.io, roughly **US$5–15/month**).

Why not a computer at home?
- **Phone calls and SMS need a public internet address** that Twilio can reach at any moment. A cloud server has one. A home computer needs an extra "tunnel" service and has to stay on, awake and online.
- Keeping it off your personal computer also limits the damage if something goes wrong.

The only real reason to prefer home is running **local models** for privacy. Since the brain is OpenRouter, that doesn't apply.

For development, we build and test on your computer (or in these Claude Code sessions), then deploy to the server.

## 4. Rough monthly cost (personal use)

| Item | Estimate |
|---|---|
| Cloud server | $5–15 |
| Phone number + calls/texts | $3–10 |
| Speech-to-text + text-to-speech | $2–10 (a few cents per minute of talking) |
| Models via OpenRouter | $10–50, depending on model choice and how much it runs on timers |
| **Total** | **~$20–85/month** |

## 5. Build phases

| Phase | You get | Main pieces |
|---|---|---|
| 1 | Chat with Jarvis on Telegram. It can search the web. | Python project, Pydantic AI, OpenRouter, Telegram bot |
| 2 | It remembers you | Markdown memory + SQLite |
| 3 | It reads your calendar and email | Composio / MCP, read-only |
| 4 | It messages you first | Scheduler: morning brief, urgent-email check |
| 5 | It has its own phone number (SMS) | Twilio number, SMS front door |
| 6 | You can call it and talk | Pipecat + Deepgram + TTS voice, fast voice model |
| 7 | It can act for you, with confirmation | Write tools, PIN/confirmation flow, action log |
| 8 | It's always on | Deploy to cloud server, monitoring, backups |

Phases 1–2 are worth doing on your computer first. Phase 5 onwards needs the server, or a tunnel.

## 6. Accounts you'll need (as we reach each phase)

- OpenRouter (phase 1)
- Telegram (phase 1)
- Composio: you already have it (phase 3)
- Twilio (phase 5)
- Deepgram and a voice provider such as Cartesia or ElevenLabs (phase 6)
- Cloud server provider (phase 8)

API keys go in a private `.env` file that is **never** committed to GitHub.

## Sources
- [Pipecat OpenRouter integration](https://docs.pipecat.ai/api-reference/server/services/llm/openrouter)
- [Pipecat phone-bot quickstart (Twilio)](https://github.com/pipecat-ai/pipecat-quickstart-phone-bot)
- [Where to put the phone line: Twilio vs LiveKit vs Pipecat](https://vadim.blog/twilio-livekit-pipecat-phone-line/)
- [Pydantic AI OpenRouter docs](https://pydantic.dev/docs/ai/models/openrouter/)
- [OpenRouter framework integrations](https://openrouter.ai/docs/guides/community/frameworks-and-integrations-overview)
- [Twilio pricing](https://www.twilio.com/en-us/pricing), [Twilio AU voice pricing](https://www.twilio.com/en-us/voice/pricing/au)
