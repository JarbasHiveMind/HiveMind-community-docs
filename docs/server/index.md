# Hub & Server

The **hub** is the central brain of a HiveMind. It accepts connections from
satellites, decides what AI backend answers them, and enforces who is allowed to do
what. This section is for whoever **runs** the hub.

!!! tip "Which hub do I want?"
    The hub's "personality" is set by one choice: **which agent plugin answers
    requests**. Pick by what you want the assistant to *be*:

    - **A full voice assistant with skills** → [OVOS Skills Hub](ovos-hub.md)
    - **A chatbot / LLM** (no skills needed) → [Persona Hub](persona-hub.md)
    - **An external AI agent** (A2A protocol) → [A2A Hub](a2a-hub.md)

    Not sure, or want the easy path? The [Admin Panel](admin-panel.md) is a web UI that
    launches a hub **and** lets you administer it from the browser — one command.

## What's in this section

| Page | What it covers |
|---|---|
| [Admin Panel](admin-panel.md) | A web UI that launches a hub and administers it — the easiest way to run one. |
| [OVOS Skills Hub](ovos-hub.md) | The default hub: serves OpenVoiceOS skills, STT, and TTS. |
| [Persona Hub (LLM)](persona-hub.md) | Serve an LLM/chatbot persona — no `ovos-core` required. |
| [A2A Hub (Agents)](a2a-hub.md) | Bridge to any agent speaking Google's Agent-to-Agent protocol. |
| [Transports](transports.md) | How bytes travel: WebSocket (default), HTTP, MQTT, Usenet. |
| [Audio Binary Protocol](audio-binary-protocol.md) | How raw audio (wake word, STT, TTS streams) travels to/from thin satellites. |
| [Docker Deployment](docker.md) | Run the hub and the OVOS stack in containers. |
| [Operations](operations.md) | Day-2 concerns: TLS, reconnection, monitoring, connection errors. |

## How a request flows through the hub

```
satellite ──▶ transport plugin ──▶ permission check ──▶ agent plugin ──▶ AI backend
 (mic, CLI)    (WebSocket/HTTP…)     (allowed? )         (OVOS/LLM/A2A)   (skills/LLM)
```

The **transport** decides *how* bytes arrive, the **permission check** decides *whether*
the message is allowed, and the **agent plugin** decides *who answers*. Each of those
three is swappable — see [Plugin Architecture](../concepts/plugins.md).

---

**Next:** [OVOS Skills Hub](ovos-hub.md) · or [Quick Start](../quickstart.md) to stand one up now.
