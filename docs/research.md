# Building a personal "Jarvis": research notes

_Researched September 2026. The field changes fast, so check each project's current docs before building on it._

## 1. The short version

- A "Jarvis" is **not one piece of software**. Every serious project combines the same five parts: **ears and voice**, a **brain**, **memory**, **hands** (tools and connections to your apps), and a **heartbeat** (a scheduler, so it can act without being asked).
- You **do not need to build or train your own AI model**. Every project below rents a model (Claude, GPT, Gemini) or runs a free open one on its own machine (for example Llama or Qwen, run through Ollama). "Building the brain" really means: **picking a model, writing its instructions, giving it memory, and giving it tools.**
- Most of the Jarvis videos online are one of two kinds:
  1. **Weekend demos**: a Python script that listens to the microphone, sends the text to ChatGPT, and reads the answer out loud. Easy to copy, but they don't remember anything and can't do much.
  2. **Setups built on a real agent platform**, mostly **OpenClaw** and more recently **Hermes Agent**. These are the ones that actually "do things": they read email, run tasks on a schedule and message you on WhatsApp or Telegram.
- **The biggest risk is security, not difficulty.** An agent that can read your email and run commands can be tricked by a malicious email or web page ("prompt injection"). OpenClaw has had several published security flaws, and malicious add-ons have been found in its community marketplace. Build with that in mind from day one.

## 2. The anatomy of a Jarvis

```
        you (voice / chat app / phone)
                    │
       ┌────────────▼────────────┐
       │  EARS + VOICE           │  speech-to-text in, text-to-speech out
       └────────────┬────────────┘
                    │ text
       ┌────────────▼────────────┐      ┌───────────────┐
       │  BRAIN (the LLM)        │◄────►│  MEMORY       │ facts about you,
       │  + instructions         │      │               │ past conversations,
       └────────────┬────────────┘      └───────────────┘ your notes
                    │ tool calls
       ┌────────────▼────────────┐
       │  HANDS (tools / MCP)    │  Gmail, Calendar, Drive, Strava, smart home...
       └─────────────────────────┘
       ┌─────────────────────────┐
       │  HEARTBEAT (scheduler)  │  "every 30 min, check if anything needs me"
       └─────────────────────────┘
```

**MCP (Model Context Protocol)** is the standard way to plug tools into an AI in 2026. Each app (Gmail, Calendar, Notion...) exposes an "MCP server". Any MCP-compatible brain can use it. This is the most important idea for a "fully connected" agent: **you don't write integrations one by one, you plug in MCP servers.**

## 3. Open-source projects worth knowing

### Full "personal agent" platforms (closest to a real Jarvis)

| Project | What it is | Good for | Watch out for |
|---|---|---|---|
| [OpenClaw](https://github.com/openclaw/openclaw) (formerly Clawdbot / Moltbot) | The project behind most of the viral "Jarvis" videos. It runs on your computer and talks to you on WhatsApp, Telegram, iMessage, Slack, Discord and more. It has voice on Mac, iOS and Android, and thousands of community "skills". | Seeing how a complete agent is put together. Its [architecture](https://dev.to/entelligenceai/inside-openclaw-how-a-persistent-ai-agent-actually-works-1mnk) is the one to copy: a background service, memory stored as Markdown files, skills written as Markdown, and a [heartbeat](https://docs.openclaw.ai/gateway/heartbeat). | Serious [security problems](https://thehackernews.com/2026/03/openclaw-ai-agent-flaws-could-enable.html): several published flaws (command injection, file reads, prompt injection) and [malicious skills in its marketplace](https://www.sangfor.com/blog/cybersecurity/openclaw-ai-agent-security-risks-2026). Microsoft published a guide on [running it safely](https://www.microsoft.com/en-us/security/blog/2026/02/19/running-openclaw-safely-identity-isolation-runtime-risk/). |
| [Hermes Agent](https://github.com/nousresearch/hermes-agent) (Nous Research) | A newer competitor to OpenClaw. It has persistent memory, 20+ chat platforms, a scheduler and MCP support, and it writes and improves its own skills over time. It can import an OpenClaw setup. | The "learns about you over time" part of Jarvis. | Newer and less proven. Self-modifying behaviour is harder to predict. |
| [Khoj](https://github.com/khoj-ai/khoj) | An open-source "second brain": it searches your documents and the web, runs scheduled automations, and works from Obsidian or WhatsApp. | Memory and knowledge over your own notes and files. | Less focused on taking actions. |
| [Leon](https://github.com/leon-ai/leon) | A long-running self-hosted assistant. Version 2.0 is being rebuilt as an agent with tools, memory and planning. | Running fully offline or privately. | 2.0 is still a developer preview. |

### "Jarvis" hobby repos (good for learning, not for daily use)

The [GitHub `jarvis` topic](https://github.com/topics/jarvis) and [`jarvis-ai` topic](https://github.com/topics/jarvis-ai) hold hundreds of these. Typical examples:
[kishanrajput23/Jarvis-Desktop-Voice-Assistant](https://github.com/kishanrajput23/Jarvis-Desktop-Voice-Assistant),
[Avinashb722/jarvis-ai-assistant](https://github.com/Avinashb722/jarvis-ai-assistant),
[rajkishorbgp/JARVIS-AI-Assistant](https://github.com/rajkishorbgp/JARVIS-AI-Assistant).
Most are a single Python file that does speech recognition, sends the text to a model and speaks the reply. They're useful for seeing the voice loop in about 100 lines. They usually have no memory, no security and no maintenance.

### Building blocks (what you'd use to build your own)

**Voice (the "ears and mouth")**
- [Pipecat](https://github.com/pipecat-ai/pipecat) and [LiveKit Agents](https://livekit.com/voice-agents) are the two most commonly recommended open-source voice frameworks. They handle the hard parts: interruptions, streaming audio and low delay. Both have quickstarts that get a talking assistant working in about 10 minutes.
- There are two ways to design the voice part:
  - **Pipeline** (speech-to-text → LLM → text-to-speech): works with any brain, including Claude. It's cheaper and easier to debug, but a bit slower.
  - **Speech-to-speech** (OpenAI Realtime, Gemini Live): feels more natural, with roughly 0.3 to 1 second response times. It ties you to that provider's model. [Comparison](https://softcery.com/lab/ai-voice-agents-real-time-vs-turn-based-tts-stt-architecture).
- **Fully local and free**: Home Assistant Assist with Whisper (speech-to-text), Ollama (brain) and Piper (text-to-speech). [Guide](https://www.kunalganglani.com/blog/local-ai-voice-assistant-whisper-piper-ollama). This is also the standard route if you want Jarvis to control lights and other smart home devices.

**Brain / agent runtime**
- [Claude Agent SDK](https://docs.claude.com/en/api/agent-sdk/overview): Anthropic's framework, the same engine that runs Claude Code. It includes the agent loop, tool use and MCP support.
- Alternatives: LangGraph, OpenAI Agents SDK, CrewAI, Smolagents ([overview](https://www.firecrawl.dev/blog/best-open-source-agent-frameworks)).
- Local models through [Ollama](https://www.home-assistant.io/integrations/ollama/) if you want privacy and no API bills. They are noticeably less capable at multi-step tasks.

**Memory**
| Tool | Approach | Best for |
|---|---|---|
| **Plain Markdown files** (the OpenClaw approach) | The agent reads and writes notes like `about-me.md` | Simplest option. You can read and edit its memory yourself. Recommended to start. |
| [Mem0](https://github.com/mem0ai/mem0) | Pulls facts out of conversations automatically | Remembering personal preferences |
| [Letta](https://github.com/letta-ai/letta) (formerly MemGPT) | The agent manages its own short-term and long-term memory | Long-running agents |
| [Graphiti / Zep](https://github.com/getzep/graphiti) | A graph of facts that tracks how they change over time | "I moved from Sydney to London". Handles facts that change. |
| [Cognee](https://www.cognee.ai/) | A graph of your knowledge, with MCP support | Connecting knowledge across many documents |

[Comparison article](https://atlan.com/know/best-ai-agent-memory-frameworks-2026/). A popular related pattern is an **Obsidian "second brain"**, a folder of Markdown notes the agent maintains ([example](https://github.com/NicholasSpisak/second-brain), based on Andrej Karpathy's "LLM wiki" idea).

**Hands (connections)**
- MCP servers exist for Gmail, Google Calendar, Drive, GitHub, Notion, Slack, Strava, Home Assistant and many more.
- [Composio](https://composio.dev/toolkits/gmail/framework/claude-agents-sdk) bundles hundreds of app connections behind one login system, so you don't have to handle each app's OAuth yourself.

## 4. Recommendation for you

You already use Claude with Gmail, Calendar, Drive, Granola, Strava, Supabase and Composio connected. Most of Jarvis's "hands" already exist for you. The missing parts are **a place for it to run 24/7, memory, a heartbeat, and a voice/chat front end.**

**Recommended path: build your own small agent with the Claude Agent SDK, copying OpenClaw's design (not its code).**

Why:
- You control exactly what it can access, which is the main security lesson from OpenClaw.
- Claude is currently among the strongest models at multi-step tool use.
- The connections you already use are MCP-based, so they carry over.
- Memory as Markdown files means you can open and correct what it "knows" about you.

**The alternative**, if you want something working this weekend: install OpenClaw or Hermes Agent on a **spare machine or a cloud server, not your main laptop**. Give it separate accounts with limited access, and only install skills you have read. You'll learn quickly what you want. But you'll be running a lot of code you didn't write, with access to your life.

### Suggested build order (each phase is usable on its own)

1. **Text Jarvis on your phone.** A Claude agent you message on Telegram, with a few tools (calendar, email read-only, web search). You learn the core loop.
2. **Memory.** An `about-me.md` file, a daily log and a notes folder that the agent reads and updates. It starts to "know" you.
3. **Heartbeat.** A scheduled job, for example a morning brief at 7am or "check my inbox every hour and only tell me if something's urgent". This is what makes it feel like Jarvis rather than a chatbot.
4. **Voice.** Add Pipecat or LiveKit so you can talk to it, with a wake word later.
5. **More hands.** Home Assistant for devices, and write actions (sending email, booking meetings) that **always ask you to confirm** first.

### Production basics (things that matter even for a personal project)

- **Hosting**: it needs to run all the time. A small cloud server (about $5 to $20/month) or a spare Mac mini at home.
- **Secrets**: keep API keys and passwords in environment variables or a secrets manager. Never put them in code or commit them to GitHub.
- **Least privilege**: start with read-only access. Add each write permission (send, delete, pay) deliberately and require confirmation.
- **Prompt injection**: assume any email or web page it reads could contain hostile instructions. Anything irreversible should need a human "yes".
- **Costs**: set a monthly spending limit on your model API account. Agents that run on a timer can quietly use a lot of tokens.
- **Logs**: record every action the agent takes so you can see what it did and why.

## 5. Open decisions for you

1. **Interface first**: phone chat (Telegram/WhatsApp), voice, or both?
2. **Cloud or local brain**: Claude (smarter, costs money, data goes to Anthropic) or a local model (private, free to run, weaker)?
3. **Where it runs**: a cheap cloud server or a computer at home?
4. **Build vs adopt**: build your own on the Claude Agent SDK (recommended), or start with OpenClaw or Hermes?

## Sources

- OpenClaw: [GitHub](https://github.com/openclaw/openclaw), [site](https://openclaw.ai/), [Wikipedia](https://en.wikipedia.org/wiki/OpenClaw), [architecture explainer](https://dev.to/entelligenceai/inside-openclaw-how-a-persistent-ai-agent-actually-works-1mnk), [Turing Post overview](https://www.turingpost.com/p/openclaw), [heartbeat docs](https://docs.openclaw.ai/gateway/heartbeat)
- OpenClaw security: [The Hacker News](https://thehackernews.com/2026/03/openclaw-ai-agent-flaws-could-enable.html), [Sangfor](https://www.sangfor.com/blog/cybersecurity/openclaw-ai-agent-security-risks-2026), [Microsoft Security Blog](https://www.microsoft.com/en-us/security/blog/2026/02/19/running-openclaw-safely-identity-isolation-runtime-risk/), [Giskard](https://www.giskard.ai/knowledge/openclaw-security-vulnerabilities-include-data-leakage-and-prompt-injection-risks), [arXiv: Taming OpenClaw](https://arxiv.org/pdf/2603.11619)
- Hermes Agent: [GitHub](https://github.com/nousresearch/hermes-agent), [self-evolution](https://github.com/NousResearch/hermes-agent-self-evolution), [awesome list](https://github.com/0xNyk/awesome-hermes-agent)
- Other assistants: [Leon](https://github.com/leon-ai/leon), [Khoj vs Leon](https://openalternative.co/compare/khoj/vs/leon), [Vellum: best open-source assistants](https://www.vellum.ai/blog/best-open-source-personal-ai-assistants)
- Voice: [Pipecat](https://github.com/pipecat-ai/pipecat), [LiveKit Agents](https://livekit.com/voice-agents), [voice framework wiki](https://soniox.com/wiki/voice-agent-frameworks), [realtime vs pipeline](https://softcery.com/lab/ai-voice-agents-real-time-vs-turn-based-tts-stt-architecture), [local voice with Home Assistant](https://www.kunalganglani.com/blog/local-ai-voice-assistant-whisper-piper-ollama), [HA Ollama integration](https://www.home-assistant.io/integrations/ollama/)
- Memory: [Atlan comparison](https://atlan.com/know/best-ai-agent-memory-frameworks-2026/), [Mem0 state of memory 2026](https://mem0.ai/blog/state-of-ai-agent-memory-2026), [Cognee](https://www.cognee.ai/best-open-source-ai-memory-tools-for-llm-agents-and-developers), [second-brain repo](https://github.com/NicholasSpisak/second-brain)
- Agent frameworks and connections: [Firecrawl framework roundup](https://www.firecrawl.dev/blog/best-open-source-agent-frameworks), [Composio + Claude Agent SDK](https://composio.dev/toolkits/gmail/framework/claude-agents-sdk)
