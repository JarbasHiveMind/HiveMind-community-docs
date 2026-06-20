# Glossary

Terms used throughout the HiveMind documentation. The user-facing guides favour
**hub** and **satellite**; the protocol and mesh references also use the
lower-level vocabulary (**node**, **Mind**, **master**/**slave**) — both describe
the same network.

## Roles

| Term | Definition |
|---|---|
| **Node** | Any device or process that participates in a HiveMind network. |
| **Hub** | A node that accepts connections, authenticates clients, isolates sessions, and authorizes individual messages. Runs the AI workload (skills, LLM, or audio processing) on behalf of its satellites. See [OVOS Skills Hub](../server/ovos-hub.md) and [Persona Hub](../server/persona-hub.md). |
| **Satellite** | A user-facing node that connects to a hub but does not accept connections of its own. Ranges from text-only ([CLI](../satellites/cli.md)) to a full local voice stack ([voice satellite](../satellites/voice-sat.md)). See [Choosing a Satellite](../satellites/index.md). |
| **Mind** | Protocol term for a node that listens for connections and understands natural-language commands — i.e. a hub. A Mind authenticates other nodes, isolates their connections, and authorizes each message. |
| **Master** | The node at the top of a hive that receives connections but does not itself connect upward as a client. |
| **Slave** | A hub that connects to another hub as a client and accepts bus messages from it; this is how [nested hives](../concepts/mesh.md#nested-hives) are formed. A satellite differs from a slave in that it is not itself a hub. |
| **Bridge** | A node that links an external service (Matrix, Mattermost, DeltaChat, Twitch, …) to a hub, relaying messages in both directions. See [Integrations](../integrations/matrix.md). |
| **Relay** | A satellite that streams audio to the hub for STT/TTS while keeping wakeword detection local. See [Voice Relay](../satellites/voice-relay.md). |

## Network structure

| Term | Definition |
|---|---|
| **Hive** | A collection of interconnected nodes — one hub and the satellites (and sub-hubs) attached to it. |
| **Nested hive** | A hub that is also a client of a parent hub, so hives stack into a hierarchy. `ESCALATE` travels up the chain, `BROADCAST` travels down. See [Mesh & Hub Topology](../concepts/mesh.md). |
| **`site_id`** | A label identifying the physical location or grouping of a node (e.g. a room or building), used for routing and addressing within a hive. |
| **Session** | The per-client context a hub maintains for each connected satellite — independent permissions, identity, and conversational state. |

## Protocol

| Term | Definition |
|---|---|
| **HiveMind protocol** | The encrypted message protocol satellites and hubs speak. Transport-agnostic; WebSocket is the default. See [Protocol & Message Types](../concepts/protocol.md) and the [Protocol Specification](../developers/protocol-spec.md). |
| **Handshake** | The connection setup that derives a shared session key (PBKDF2-HMAC-SHA256, 100 000 iterations) and establishes an encrypted channel using ChaCha20-Poly1305 or AES-GCM, backed by per-node RSA identities. See [Security & Permissions](../concepts/security.md). |
| **BUS message** | An OVOS `messagebus` message carried over the hive, the primary payload type for utterances and responses. |
| **Bus message types** | The transport verbs that route messages through the mesh: `ESCALATE`, `BROADCAST`, `PROPAGATE`, `QUERY`, `CASCADE`, `INTERCOM`. See [Message Types](message-types.md). |
| **Binary protocol** | The framing used to carry raw audio (and other binary payloads) over the hive for server-side STT/TTS. See [Audio Binary Protocol](../server/audio-binary-protocol.md). |
| **Permission / policy** | The deny-by-default rules that decide which message types and skills a given client may use, enforced as a policy chain. See [Security & Permissions](../concepts/security.md). |

## Discovery

| Term | Definition |
|---|---|
| **HiveBeacon** | A planned zero-dependency UDP-broadcast auto-discovery transport, intended to become the default once it ships. Not yet implemented — the current default LAN discovery transport is mDNS/Zeroconf (with UPnP/SSDP as an off-by-default legacy option). See [Auto Discovery](../concepts/discovery.md). |
| **GGWave pairing** | An audio-based pairing mechanism that exchanges connection details over sound. Proof-of-concept. See [Auto Discovery](../concepts/discovery.md). |

## OVOS & voice vocabulary

Terms a HiveMind user runs into that come from the OVOS voice stack a hub runs.

| Term | Definition |
|---|---|
| **OVOS (OpenVoiceOS)** | The open-source voice assistant platform a HiveMind hub runs skills on. See [openvoiceos.org](https://openvoiceos.org). |
| **ovos-core** | The skills/intent engine — the default HiveMind hub agent. |
| **messagebus / ovos-messagebus** | OVOS's internal websocket event bus that skills and services communicate over locally. It is distinct from the HiveMind protocol, which is what a hub and its satellites speak; a BUS message carries an OVOS `messagebus` message across the hive. |
| **Skill** | A plugin that handles a category of intents (weather, timers, etc.). |
| **Intent** | The parsed meaning of an utterance, used to route it to a skill. |
| **Utterance** | A transcribed line of user speech (or typed text). |
| **Wakeword** | The trigger phrase (e.g. "hey mycroft") that starts listening. |
| **VAD (Voice Activity Detection)** | Detects when speech starts and stops in the microphone stream. |
| **STT (Speech-to-Text)** | Transcribes audio into text. |
| **TTS (Text-to-Speech)** | Synthesizes spoken audio from text. |
| **Solver** | An `ovos-persona` plugin that answers a query (e.g. an LLM backend); personas chain solvers together. |
| **Persona** | An `ovos-persona` config — a named chain of solvers — that a [Persona Hub](../server/persona-hub.md) serves instead of full skills. |
| **OCP (OVOS Common Play)** | The OVOS media/audio playback framework. |
| **PHAL** | OVOS's Platform Hardware Abstraction Layer — plugins for volume, system control, and similar; relevant to the Home Assistant integration's permission list. |
