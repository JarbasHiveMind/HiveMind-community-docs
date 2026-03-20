# What is HiveMind?

HiveMind is an open-source protocol and platform that connects lightweight satellite devices to a central AI hub over the network — including hardware that cannot run a full AI system on its own.

## The core idea

HiveMind separates the AI workload from edge devices. Satellites (microphones, voice devices, browsers, chat clients) delegate processing to a central hub, which handles reasoning, skills, and audio processing.

The hub can be one of two types:

**OVOS Hub (Skills Server)**: Run [OpenVoiceOS](https://openvoiceos.github.io/community-docs) on a capable machine (desktop, home server, Raspberry Pi 4) and connect satellites over the network:
```
[Mic Satellite]──┐
[Voice Relay]────┤──→ [HiveMind Hub (OVOS)] ──→ skills, intents, TTS
[Voice Satellite]┘
```

**Persona Hub (LLM/Chatbot Server)**: Run a standalone LLM or chatbot without OVOS:
```
[Browser Client]──┐
[Voice Device]────┤──→ [HiveMind Hub (LLM)] ──→ responses, personas, embeddings
[Chat Client]─────┘
```

Satellites can be anything that speaks the HiveMind protocol: dedicated voice hardware, a browser tab, a chat room, a Home Assistant instance, or a custom Python client.

## What the hub can be

| Hub type | Package | Use case |
|---|---|---|
| OVOS skills server | `hivemind-core` | Full OVOS with installed skills |
| Sound server (listener) | `hivemind-listener` | Offloads STT/TTS/WakeWord from satellites |
| Persona server | `hivemind-persona` | LLMs and chatbots via `ovos-persona` |

## What connects to the hub

| Client type | Package | What runs locally |
|---|---|---|
| Voice Satellite | `HiveMind-voice-sat` | Full local STT, TTS, WakeWord |
| Voice Relay | `HiveMind-voice-relay` | WakeWord + VAD only, STT/TTS on hub |
| Mic Satellite | `hivemind-mic-satellite` | Microphone + VAD only, everything on hub |
| Web browser | `hivemind-webspeech` | JavaScript VAD, audio streamed to hub |
| Home Assistant | `hivemind-homeassistant` | OCP media player and system control |
| Matrix room | `HiveMind-matrix-bridge` | Chat messages bridged to OVOS |
| DeltaChat (email) | `HiveMind-deltachat-bridge` | Email messages bridged to OVOS |
| Python script | `hivemind-websocket-client` | Direct protocol access |

## Security model

Every client authenticates with an **access key** and a **password**. After authentication the session is encrypted with AES-256-GCM. Permissions are configured per client — you can allow or deny individual OVOS message types, skills, and intents for each connected device.

Protocol v0 (legacy) supports a pre-shared encryption key only. Protocol v1 adds a PBKDF2 password handshake, RSA public-key identity, zlib compression, and binary serialization.

## Nested hives

HiveMind hubs can connect to other HiveMind hubs, forming a hierarchy. A sub-hub acts as a slave to its parent while being a master to its own satellites. `ESCALATE` messages travel up the chain; `BROADCAST` messages travel down. This enables multi-household or multi-room topologies where each hub controls its local devices but can route queries upward.

## What HiveMind is not

- It is **not** a replacement for OVOS when running as a skills server. For the `hivemind-core` hub type, OVOS remains the AI back-end. The `hivemind-persona` hub type runs without OVOS.
- It is **not** a cloud service. Everything runs on your own hardware.
- It does **not** replace the OVOS messagebus for local communication. The messagebus remains internal; HiveMind is the external-facing gateway.
