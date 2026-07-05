# Frequently Asked Questions

**This page gives short answers with links to the full pages.** New to the vocabulary? Keep the
[Glossary](reference/glossary.md) open in another tab.

!!! abstract "In a nutshell"
    - Quick answers grouped by topic: general concepts, setup, security, satellites, integration, and troubleshooting.
    - Each answer links to the page that covers it in depth.
    - Start with the [Quick Start](quickstart.md) or [Admin Panel](server/admin-panel.md) to get hivemind-core running.

## General

??? question "What is HiveMind?"
    HiveMind is an open-source protocol and platform that connects lightweight
    **[satellite](reference/glossary.md#roles)** devices (a microphone, a browser tab, a
    terminal) to a central AI server — **[hivemind-core](reference/glossary.md#roles)** — over an encrypted
    network. hivemind-core does the heavy lifting — skills, reasoning, speech-to-text — so the
    satellites can be cheap and simple. See [About](about.md).

??? question "How is it different from a single-device voice assistant?"
    The work is **distributed**. One capable hivemind-core server serves many lightweight satellites, each
    with its own session and permissions. Speech processing can happen on the satellite or
    on the server, depending on which satellite you choose ([comparison](satellites/index.md)).

??? question "Which AI backend does it use?"
    HiveMind is **backend-agnostic**. By default hivemind-core runs
    [OpenVoiceOS](reference/glossary.md#ovos-voice-vocabulary) skills, but you can instead
    point it at an LLM/chatbot ([Persona Server](server/persona-hub.md)) or an external agent
    ([A2A Server](server/a2a-hub.md)). The choice is just *which
    [agent plugin](concepts/plugins.md)* hivemind-core loads.

??? question "Do satellites talk directly to each other?"
    No. Satellites connect to **hivemind-core**, which routes everything. Two clients *can*
    exchange end-to-end-encrypted **[INTERCOM](concepts/security.md)** messages, but those
    are still relayed through hivemind-core — there is no satellite-to-satellite peer link.
    hivemind-core instances, on the other hand, can connect to *other instances* to form a
    [mesh](concepts/mesh.md).

---

## Setup

??? question "What's the easiest way to get started?"
    Two options:

    - **Web UI:** the [Admin Panel](server/admin-panel.md) starts hivemind-core *and* gives you a
      browser to manage it — one command.
    - **Command line:** the [Quick Start](quickstart.md) takes you from zero to a working
      hivemind-core + voice satellite in about ten minutes.

    No hardware? Try the [WebSpeech browser satellite](satellites/webspeech.md).

??? question "Do I need a dedicated machine for hivemind-core?"
    No. hivemind-core runs fine on a laptop/desktop (Linux, macOS, Windows via WSL), a Raspberry
    Pi 4+ (good for several satellites), a cloud VM, or a [Docker container](server/docker.md).

??? question "Can I run it over the internet?"
    Yes, but secure it properly. On a LAN, use a strong password (and optionally self-signed
    TLS). Internet-facing, put it behind a reverse proxy (Caddy, nginx, Traefik) that
    terminates real TLS, and keep a strong password + firewall. See
    [Operations](server/operations.md) and [Security](concepts/security.md).

---

## Security

??? question "Is HiveMind secure?"
    Yes. Connections use authenticated encryption (AES-256-GCM or ChaCha20-Poly1305,
    negotiated), with a password-based handshake (PBKDF2-HMAC-SHA256, 100 000 iterations).
    The encryption key is never sent over the wire. Permissions are **fail-closed**: a new
    client may do nothing until you grant it specific message types. Full detail in
    [Security & Permissions](concepts/security.md).

??? question "WebSocket vs HTTP — what's the difference?"
    **WebSocket** (default, port 5678) is a persistent, low-latency, bidirectional
    connection — ideal for real-time voice. **HTTP** is request/response, simpler for
    polling/REST-style clients. Both carry the exact same encrypted messages. See
    [Transports](server/transports.md).

---

## Satellites & devices

??? question "Which satellite should I choose?"
    Quick guide (full table in [Choosing a Satellite](satellites/index.md)):

    - **Text only / no audio** → [HiveMind CLI](satellites/cli.md)
    - **Cheapest mic device** → [Mic Satellite](satellites/mic-satellite.md)
    - **Raspberry Pi voice** → [Voice Relay](satellites/voice-relay.md) or [Voice Satellite](satellites/voice-sat.md)
    - **Any browser** → [WebSpeech](satellites/webspeech.md)
    - **Microcontroller (ESP32)** → [Microcontrollers](satellites/microcontrollers.md)

??? question "Can I have multiple satellites?"
    Yes. Each connects independently with its own access key and permissions. They can share
    an OVOS session for conversational continuity.

---

## Integration & development

??? question "Can I run OVOS skills?"
    Yes — the OVOS skills backend runs the full skill stack; existing OVOS skills work as-is.
    See [OVOS Skills Server](server/ovos-hub.md).

??? question "How do I connect a chat platform or smart home?"
    Use a [bridge](integrations/index.md): [Home Assistant](integrations/home-assistant.md),
    [Matrix](integrations/matrix.md), [Twitch](integrations/twitch.md),
    [Mattermost](integrations/mattermost.md), [DeltaChat](integrations/deltachat.md),
    [HackChat](integrations/hackchat.md). Or write your own with the
    [Client Library](developers/client-library.md).

??? question "How do I write a custom satellite?"
    Use the [Client Library](developers/client-library.md) (`HiveMessageBusClient` in
    Python) or [HiveMind-js](satellites/webspeech.md) for the browser. To implement the
    protocol from scratch, follow the [Protocol Specification](developers/protocol-spec.md).

---

## Troubleshooting

??? question "A satellite can't connect to hivemind-core"
    1. Confirm hivemind-core is running and reachable (`ping <server-ip>`).
    2. From the satellite, test the credentials: `hivemind-client test-identity`.
    3. Check the firewall allows port 5678 (WebSocket) or 5679 (HTTP).
    4. Make sure the client is **allowed** — a brand-new client is denied every message type
       until you run `hivemind-core allow-msg ...` (see [Security](concepts/security.md#permissions)).

??? question "How do I find the server's address automatically?"
    Run `hivemind-presence scan` on the same network (hivemind-core announces itself when auto
    discovery is enabled). Most satellites also auto-scan when you start them without a
    `--host`. See [Auto Discovery](concepts/discovery.md).

??? question "Audio is choppy or delayed"
    Prefer wired Ethernet or 5 GHz Wi-Fi; move wakeword/VAD onto the satellite
    ([Voice Relay](satellites/voice-relay.md) or [Voice Satellite](satellites/voice-sat.md)
    rather than [Mic Satellite](satellites/mic-satellite.md)); and check hivemind-core isn't
    CPU-bound.

---

## Documentation

??? question "How do I build these docs locally?"
    ```bash
    pip install -r requirements.txt
    mkdocs serve
    ```
    Then open <http://localhost:8000>. Contributions welcome — everything is Markdown under
    `docs/`, published automatically to
    [GitHub Pages](https://jarbashivemind.github.io/HiveMind-community-docs/).
