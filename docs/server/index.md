# hivemind-core Server

Everything in a HiveMind points back to one machine: the server in the closet, on the
desk, in the container. hivemind-core is what accepts every satellite's connection,
decides which AI actually answers, and holds the line on who's allowed to do what. The
satellites are interchangeable; this is the piece that makes them a hive. This section
is for whoever **runs** it.

!!! tip "Which server do I want?"
    A server's "personality" is set by one choice: **which agent plugin answers
    requests**. Pick by what you want the assistant to *be*:

    - **A full voice assistant with skills** → [OVOS Skills Server](ovos-server.md)
    - **A chatbot / LLM** (no skills needed) → [Persona Server](persona-server.md)
    - **An external AI agent** (A2A protocol) → [A2A Server](a2a-server.md)

    Not sure, or want the easy path? The [Admin Panel](admin-panel.md) is a web UI that
    launches a `hivemind-core` instance **and** lets you administer it from the browser — one command.

---

## What's in this section

| Page | What it covers |
|---|---|
| [Admin Panel](admin-panel.md) | A web UI that launches a `hivemind-core` instance and administers it — the easiest way to run one. |
| [OVOS Skills Server](ovos-server.md) | The default setup: serves OpenVoiceOS skills, STT, and TTS. |
| [Persona Server (LLM)](persona-server.md) | Serve an LLM/chatbot persona — no `ovos-core` required. |
| [A2A Server (Agents)](a2a-server.md) | Bridge to any agent speaking Google's Agent-to-Agent protocol. |
| [Transports](transports.md) | How bytes travel: WebSocket (default), HTTP, MQTT, Usenet. |
| [Audio Binary Protocol](audio-binary-protocol.md) | How raw audio (wake word, STT, TTS streams) travels to/from thin satellites. |
| [Docker Deployment](docker.md) | Run `hivemind-core` and the OVOS stack in containers. |
| [Operations](operations.md) | Day-2 concerns: TLS, reconnection, monitoring, connection errors. |

---

## How a request flows through hivemind-core

Whichever flavour you pick, every question a satellite asks takes the same short journey
through the server. Follow one utterance left to right:

```
satellite ──▶ transport plugin ──▶ permission check ──▶ agent plugin ──▶ AI backend
 (mic, CLI)    (WebSocket/HTTP…)     (allowed? )         (OVOS/LLM/A2A)   (skills/LLM)
```

Three checkpoints, three questions: the **transport** decides *how* the bytes arrive, the
**permission check** decides *whether* the message is allowed through, and the **agent
plugin** decides *who answers* it. What makes hivemind-core hivemind-core is that all
three are swappable parts, not fixed wiring — which is the whole story of
[Plugin Architecture](../concepts/plugins.md).

---

**Next:** [OVOS Skills Server](ovos-server.md) · or [Quick Start](../quickstart.md) to stand one up now.
