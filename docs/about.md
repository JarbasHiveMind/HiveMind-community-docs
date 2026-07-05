# What is HiveMind?

**HiveMind is an open-source protocol and platform** that connects lightweight **satellite** devices to a central AI server — **hivemind-core** — over the network, including hardware that cannot run a full AI system on its own.

!!! abstract "In a nutshell"
    - Satellites delegate reasoning, skills, and audio processing to hivemind-core, which serves many of them with independent sessions and permissions.
    - Configure hivemind-core with an OVOS skills backend for home automation and skills, or a persona backend to just chat with an LLM.
    - Satellites range from text-only CLI clients to full local voice stacks; hivemind-core instances can chain into a mesh.
    - HiveMind is a gateway around OVOS, not a replacement for it, and runs entirely on your own hardware.

## The core idea

HiveMind separates AI workload from edge devices. Satellites (microphones, voice devices, browsers, chat clients) delegate processing to hivemind-core, which handles reasoning, skills, and audio processing. hivemind-core can serve many satellites simultaneously with independent sessions and permissions for each.

New here? The [Glossary](reference/glossary.md) defines every term on this page.

```
[Mic Satellite]  ──┐
[Voice Relay]    ───┤──→ [hivemind-core] ──→ skills / LLM / intents
[Voice Satellite]──┤         │
[CLI Client]     ───┘        └──→ responses back to each satellite
```

---

## Backend types

Pick the **[OVOS](reference/glossary.md#ovos-voice-vocabulary) skills backend** if you want skills/home-automation; pick the **[Persona](reference/glossary.md#ovos-voice-vocabulary) backend** if you just want to chat with an LLM and want the simplest setup (no OVOS required).

| Backend | Package | Agent backend |
|---|---|---|
| OVOS skills server | `hivemind-core` | OpenVoiceOS (full skill ecosystem) |
| Audio server | `hivemind-core` + `hivemind-audio-binary-protocol` | OVOS + server-side [STT](reference/glossary.md#ovos-voice-vocabulary)/[TTS](reference/glossary.md#ovos-voice-vocabulary)/wakeword |
| Persona server | `hivemind-core` + `hivemind-persona-agent-plugin` | LLMs and chatbots via `ovos-persona` |

---

## Satellite types

Satellites form a spectrum based on where audio processing happens. At one end the satellite does nothing but pass text; at the other it handles the complete voice stack locally.

| Satellite | Local processing | Server requirements |
|---|---|---|
| **HiveMind-cli** | Text I/O only | Any hivemind-core |
| **hivemind-mic-satellite** | Mic + [VAD](reference/glossary.md#ovos-voice-vocabulary) | Audio binary protocol for STT/TTS/wakeword |
| **HiveMind-voice-relay** | Mic + VAD + Wakeword | Audio binary protocol for STT/TTS |
| **HiveMind-voice-sat** | Mic + VAD + Wakeword + STT + TTS | Any hivemind-core (sends text utterances) |
| **hivemind-webspeech** | Browser mic + VAD | hivemind-core listener + `allow-msg` (see page) |
| **ESP32 / MicroPython** | Mic (and, on some boards, VAD/wakeword) | Audio binary protocol |

See [Choosing a Satellite](satellites/index.md) for a detailed comparison.

---

## Mesh topology

HiveMind-core instances can connect to other instances, forming a hierarchy. A child instance acts as a client to its parent while being a server to its own satellites. `ESCALATE` messages travel up the chain; `BROADCAST` messages travel down. This enables multi-room or multi-household topologies where each instance controls its local devices but can route requests upward.

```
[Root hivemind-core]
    ├──→ [Child A] ──→ satellites
    └──→ [Child B] ──→ satellites
```

See [Nested Hives](concepts/mesh.md#nested-hives) for details.

---

## Security model

Every client authenticates with an **access key** and a **password**. After authentication the session is encrypted with an authenticated cipher — AES-256-GCM or ChaCha20-Poly1305, negotiated per connection. The key is never transmitted; both sides derive it independently via PBKDF2 from the shared password and exchanged nonces.

Permissions are configured per client. The default policy admits only a basic set of message types; you expand or restrict per-client using the `hivemind-core` CLI. Skills and intents can be blacklisted individually.

See [Security](concepts/security.md) for the full model.

---

## What HiveMind is not

- It is **not** a replacement for OVOS when running as a skills server. OVOS remains the AI back-end; HiveMind is the external-facing gateway.
- It is **not** a cloud service. Everything runs on your own hardware.
- It does **not** replace the OVOS messagebus for local communication. The messagebus remains internal.

---

**Next:** [Quick Start](quickstart.md) to set up your first hivemind-core and satellite, or [Core Concepts](concepts/protocol.md) for how the protocol works.
