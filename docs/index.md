
# HiveMind

![HiveMind Logo](https://github.com/JarbasHiveMind/HiveMind-assets/raw/master/logo/hivemind-512.png)

HiveMind is an open-source protocol and platform that connects satellite devices to a central AI hub — either an [OpenVoiceOS](https://openvoiceos.github.io/community-docs) skills server or a standalone LLM/persona server.

- **One hub, many satellites** — run OVOS, an LLM, or a chatbot on your hub machine. Every other device connects as a satellite over the network.
- **Secure by default** — all traffic is AES-256-GCM encrypted with per-client authentication and fine-grained permissions.
- **Flexible** — voice satellites, microphone-only devices, LLM chatbots, Home Assistant, Matrix rooms, and custom clients all speak the same protocol.

---

## Where to start

| I want to… | Go to |
|---|---|
| Understand the core concepts | [What is HiveMind](about.md) |
| Set up a hub and connect a first satellite | [Quick Start](01_quickstart.md) |
| Pick the right satellite for my device | [Choosing a Satellite](satellite_comparison.md) |
| Run HiveMind with Docker | [Docker Deployment](docker.md) |
| Connect a chatbot or LLM | [Persona Server](08_persona.md) |
| Integrate with Home Assistant | [Home Assistant](07_homeassistant.md) |
| Build a client application | [Client Libraries](11_devs.md) |

---

## Community

- [Matrix chat](https://matrix.to/#/#jarbashivemind:matrix.org) — news, support, and general discussion
- [GitHub](https://github.com/JarbasHiveMind) — source code and issues
