
# HiveMind

![HiveMind Logo](https://github.com/JarbasHiveMind/HiveMind-assets/raw/master/logo/hivemind-512.png)

!!! warning "Pre-release software"
    HiveMind is under active development. Expect bugs and breaking changes between releases.

**HiveMind is an open-source protocol and platform** that connects lightweight **[satellite](reference/glossary.md#roles)** devices to a central AI server — **[hivemind-core](reference/glossary.md#roles)** — over the network. Satellites range from text-only CLI clients to full voice-capable devices. hivemind-core can run an [OpenVoiceOS](https://openvoiceos.github.io/community-docs) skills server or a standalone LLM/persona server.

!!! abstract "In a nutshell"
    - HiveMind moves the heavy AI work onto a central hivemind-core server so satellite devices can stay cheap and simple.
    - Every connection is encrypted and permissioned; each client authenticates with its own access key and password.
    - The protocol is transport- and payload-agnostic, and hivemind-core instances can chain into a mesh.

!!! tip "New to HiveMind?"
    Start with the [Glossary](reference/glossary.md) — it defines every term used here
    in one page — then follow the [Quick Start](quickstart.md). You don't need to be a
    developer to run HiveMind.

```
[Voice Satellite] ──┐
[Mic Satellite]  ───┤──→ [hivemind-core] ──→ skills / LLM / intents
[CLI Client]     ───┤
[Browser Tab]    ───┘
```

**Key properties:**

- **Transport-agnostic** — protocol plugins handle WebSocket, HTTP, MQTT, and more; swap without changing clients
- **Payload-agnostic** — OVOS messages by default; agent protocol plugins support other backends
- **Encrypted by default** — authenticated encryption (AES-256-GCM or ChaCha20-Poly1305) with per-client access keys and a PBKDF2 password handshake
- **Fine-grained ACL** — per-client allowlists for message types, skills, and intents; fail-closed
- **Mesh-capable** — hivemind-core instances connect to other instances; ESCALATE goes up the chain, BROADCAST goes down

---

## Where to start

Choose the path that fits you:

=== "I want to *use* HiveMind"

    You don't need to write any code. Most people start with the **voice satellite**
    (full local voice) or the **mic satellite** (cheapest hardware).

    | I want to… | Go to |
    |---|---|
    | Understand what HiveMind is, in plain terms | [About](about.md) |
    | Set up hivemind-core and connect my first satellite | [Quick Start](quickstart.md) |
    | Run everything from a friendly web UI | [Admin Panel](server/admin-panel.md) |
    | Pick the right satellite for my hardware | [Choosing a Satellite](satellites/index.md) |
    | Run HiveMind in Docker | [Docker Deployment](server/docker.md) |
    | Just chat with an LLM (no skills) | [Persona Server](server/persona-server.md) |
    | Integrate with Home Assistant | [Home Assistant](integrations/home-assistant.md) |
    | Get unstuck | [FAQ](faq.md) · [Glossary](reference/glossary.md) |

=== "I want to *build* on HiveMind"

    Connect from code, implement the protocol elsewhere, or extend hivemind-core.

    | I want to… | Go to |
    |---|---|
    | Connect to a hive from Python | [Client Library](developers/client-library.md) |
    | Implement HiveMind in another language | [Protocol Specification](developers/protocol-spec.md) |
    | Add a transport, backend, database, or policy | [Writing Plugins](developers/writing-plugins.md) |
    | Test a satellite, hivemind-core, or whole mesh | [Testing](developers/testing.md) |
    | Look up a CLI flag, config key, or message type | [Reference](reference/index.md) |

---

## Community

- [Matrix chat](https://matrix.to/#/#jarbashivemind:matrix.org) — news, support, and general discussion
- [GitHub](https://github.com/JarbasHiveMind) — source code and issues
- [YouTube channel](https://www.youtube.com/channel/UCYoV5kxp2zrH6pnoqVZpKSA/) — video guides and demos

---

**Next:** [Quick Start](quickstart.md) — set up hivemind-core and connect your first satellite.
