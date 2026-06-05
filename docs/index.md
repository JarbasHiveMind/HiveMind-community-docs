
# HiveMind

![HiveMind Logo](https://github.com/JarbasHiveMind/HiveMind-assets/raw/master/logo/hivemind-512.png)

HiveMind is an open-source protocol and platform that connects lightweight **satellite** devices to a central AI **hub** over the network. Satellites range from text-only CLI clients to full voice-capable devices. The hub can be an [OpenVoiceOS](https://openvoiceos.github.io/community-docs) skills server or a standalone LLM/persona server.

```
[Voice Satellite] ──┐
[Mic Satellite]  ───┤──→ [HiveMind Hub] ──→ skills / LLM / intents
[CLI Client]     ───┤
[Browser Tab]    ───┘
```

**Key properties:**

- **Transport-agnostic** — protocol plugins handle WebSocket, HTTP, MQTT, and more; swap without changing clients
- **Payload-agnostic** — OVOS messages by default; agent protocol plugins support other backends
- **Encrypted by default** — AES-256-GCM with per-client access keys and PBKDF2 password handshake
- **Fine-grained ACL** — per-client allowlists for message types, skills, and intents; fail-closed
- **Mesh-capable** — hubs connect to other hubs; ESCALATE goes up the chain, BROADCAST goes down

---

## Where to start

| I want to… | Go to |
|---|---|
| Understand what HiveMind is | [About](about.md) |
| Set up a hub and connect my first satellite | [Quick Start](quickstart.md) |
| Pick the right satellite for my hardware | [Choosing a Satellite](satellites/index.md) |
| Run HiveMind in Docker | [Docker Deployment](server/docker.md) |
| Connect a chatbot or LLM | [Persona Hub](server/persona-hub.md) |
| Integrate with Home Assistant | [Home Assistant](integrations/home-assistant.md) |
| Build a client application | [Developer Guide](developers/client-library.md) |
| Read the protocol specification | [Protocol Reference](developers/protocol-spec.md) |

---

## Community

- [Matrix chat](https://matrix.to/#/#jarbashivemind:matrix.org) — news, support, and general discussion
- [GitHub](https://github.com/JarbasHiveMind) — source code and issues
- [YouTube channel](https://www.youtube.com/channel/UCYoV5kxp2zrH6pnoqVZpKSA/) — video guides and demos
