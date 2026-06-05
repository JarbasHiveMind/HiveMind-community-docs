# What is HiveMind?

HiveMind is an open-source protocol and platform that connects lightweight **satellite** devices to a central AI **hub** over the network — including hardware that cannot run a full AI system on its own.

## The core idea

HiveMind separates AI workload from edge devices. Satellites (microphones, voice devices, browsers, chat clients) delegate processing to a hub, which handles reasoning, skills, and audio processing. The hub can serve many satellites simultaneously with independent sessions and permissions for each.

```
[Mic Satellite]  ──┐
[Voice Relay]    ───┤──→ [HiveMind Hub] ──→ skills / LLM / intents
[Voice Satellite]──┤         │
[CLI Client]     ───┘        └──→ responses back to each satellite
```

## Hub types

| Hub | Package | Agent backend |
|---|---|---|
| OVOS skills server | `hivemind-core` | OpenVoiceOS (full skill ecosystem) |
| Audio hub | `hivemind-core` + `hivemind-audio-binary-protocol` | OVOS + server-side STT/TTS/wakeword |
| Persona server | `hivemind-persona` | LLMs and chatbots via `ovos-persona` |

## Satellite types

Satellites form a spectrum based on where audio processing happens. At one end the satellite does nothing but pass text; at the other it handles the complete voice stack locally.

| Satellite | Local processing | Hub requirements |
|---|---|---|
| **HiveMind-cli** | Text I/O only | Any hub |
| **hivemind-mic-satellite** | Mic + VAD | Audio binary protocol for STT/TTS/wakeword |
| **HiveMind-voice-relay** | Mic + VAD + Wakeword | Audio binary protocol for STT/TTS |
| **HiveMind-voice-sat** | Mic + VAD + Wakeword + STT + TTS | Any hub (sends text utterances) |
| **hivemind-webspeech** | Browser mic + VAD | Any hub |

See [Choosing a Satellite](satellites/index.md) for a detailed comparison.

## Mesh topology

HiveMind hubs can connect to other hubs, forming a hierarchy. A sub-hub acts as a client to its parent while being a server to its own satellites. `ESCALATE` messages travel up the chain; `BROADCAST` messages travel down. This enables multi-room or multi-household topologies where each hub controls its local devices but can route requests upward.

```
[Master Hub]
    ├──→ [Sub-hub A] ──→ satellites
    └──→ [Sub-hub B] ──→ satellites
```

See [Nested Hives](concepts/mesh.md#nested-hives) for details.

## Security model

Every client authenticates with an **access key** and a **password**. After authentication the session is encrypted with AES-256-GCM. The key is never transmitted; both sides derive it independently via PBKDF2 from the shared password and exchanged nonces.

Permissions are configured per client. The default policy admits only a basic set of message types; you expand or restrict per-client using the `hivemind-core` CLI. Skills and intents can be blacklisted individually.

See [Security](concepts/security.md) for the full model.

## What HiveMind is not

- It is **not** a replacement for OVOS when running as a skills server. OVOS remains the AI back-end; HiveMind is the external-facing gateway.
- It is **not** a cloud service. Everything runs on your own hardware.
- It does **not** replace the OVOS messagebus for local communication. The messagebus remains internal.
