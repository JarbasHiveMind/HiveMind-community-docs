# HiveMind Community Docs — Frequently Asked Questions

## General Questions

**Q: What is HiveMind?**

A: HiveMind is a decentralized, mesh-networking AI framework that connects lightweight satellite devices to a central AI hub. It enables voice assistants, chatbots, and AI services to run across multiple devices with secure, encrypted communication.

**Q: How is HiveMind different from single-device voice assistants?**

A: HiveMind enables distributed AI:
- Voice processing can happen locally (on satellites) or remotely (on the hub)
- Satellites are lightweight; the hub does heavy lifting
- Multiple satellites can work together through a mesh network
- Peer-to-peer encrypted communication between satellites

**Q: Which AI backend does HiveMind use?**

A: HiveMind is **AI-backend-agnostic**. It ships with OpenVoiceOS (OVOS) by default, but the protocol can work with any AI system: GPT, Claude, Ollama, or custom solvers.

---

## Setup & Installation

**Q: What's the easiest way to get started?**

A: Start with [Quick Start](/HiveMind-community-docs/docs/01_quickstart.md).

For a first hub: Use Docker — see [Docker Deployment](/HiveMind-community-docs/docs/docker.md).

For a first satellite: Try the [WebSpeech Browser Satellite](/HiveMind-community-docs/docs/webspeech.md) (no hardware required).

**Q: Do I need a dedicated server for the hub?**

A: No. The hub can run on:
- Laptop/desktop (Linux, macOS, Windows with WSL)
- Raspberry Pi 4+ (recommended for 2+ satellites)
- VPS/cloud VM (for remote setup)
- Docker container (simplest)

**Q: What are the hardware requirements?**

A: **Hub**: 2GB RAM, dual-core CPU minimum. **Satellites**: Highly variable (see [Choosing a Satellite](/HiveMind-community-docs/docs/satellite_comparison.md)).

---

## Protocols & Security

**Q: Is HiveMind secure?**

A: Yes. HiveMind uses:
- AES-256-GCM encryption (confidentiality + integrity)
- Password-based handshakes with PBKDF2 key derivation (100k iterations)
- Optional SSL/TLS at the transport layer
- See [Encryption](/HiveMind-community-docs/docs/19_crypto.md) and [Handshake](/HiveMind-community-docs/docs/12_handshake.md) for details.

**Q: Can I use HiveMind over the internet?**

A: Yes, but best practices differ by deployment:

**Local Network**: Use native SSL with self-signed certs + strong password

**Internet-Facing**: Use a reverse proxy (nginx proxy manager, Caddy, Traefik):
- Proxy handles valid TLS certificates (Let's Encrypt)
- HiveMind stays on private/internal port
- Proxy provides rate limiting, load balancing, WAF
- Strong password + firewall still required (defense in depth)

See [Security Best Practices](/HiveMind-community-docs/docs/19_crypto.md#network-security)

**Q: What's the difference between the WebSocket and HTTP transports?**

A:
- **WebSocket** (port 5678): Persistent bidirectional connection; low latency; ideal for real-time voice
- **HTTP** (port 5679): Request-response; higher latency; ideal for polling devices, REST clients

See [HTTP Transport](/HiveMind-community-docs/docs/http_protocol.md) for details.

---

## Satellites & Devices

**Q: Which satellite should I choose?**

A: See [Choosing a Satellite](/HiveMind-community-docs/docs/satellite_comparison.md) for a detailed comparison. Quick guide:
- **Minimal IoT** → Mic Satellite
- **Raspberry Pi** → Voice Satellite or Voice Relay
- **Any browser** → WebSpeech Browser Satellite
- **Always-on listening** → WebSpeech or Voice Satellite with WakeWord mode

**Q: Can I have multiple satellites?**

A: Yes! Each satellite connects independently to the hub. They can share the same OVOS session for conversational continuity.

**Q: What's the network bandwidth for a voice satellite?**

A: Approximately **16-64 kbps** for audio STT (16 kHz mono, 16-bit PCM). Encrypted with AES, slightly higher.

---

## Skills & Integration

**Q: Can I run OVOS skills on HiveMind?**

A: Yes! HiveMind runs the full OVOS skill stack. Any OVOS skill works out of the box.

**Q: How do I integrate HiveMind with external systems (Mattermost, Home Assistant, etc.)?**

A: Use a bridge plugin:
- [Mattermost](/HiveMind-community-docs/docs/mattermost.md)
- [Twitch](/HiveMind-community-docs/docs/twitch.md)
- [Matrix](/HiveMind-community-docs/docs/09_matrix.md)
- [DeltaChat](/HiveMind-community-docs/docs/10_deltachat.md)
- [Home Assistant](/HiveMind-community-docs/docs/07_homeassistant.md)

Or write a custom bridge using the [Client Libraries](/HiveMind-community-docs/docs/11_devs.md).

**Q: Can I use custom AI solvers instead of OVOS skills?**

A: Yes. Use `ovos-persona` to integrate external AI backends (GPT, Claude, Ollama, etc.). See [Persona Server](/HiveMind-community-docs/docs/08_persona.md).

---

## Networking & Discovery

**Q: How do I find the hub's address?**

A: Use automatic discovery:
```bash
hivemind-client discover
```

Or manually specify in identity file: `~/.config/hivemind-core/identity.json`

See [Network Discovery](/HiveMind-community-docs/docs/20_network_discovery.md).

**Q: What happens if a satellite loses connection?**

A: The satellite will attempt automatic reconnection. Undelivered messages are queued on the hub (for HTTP clients) or lost (for WebSocket). See [Presence & Discovery](/HiveMind-community-docs/docs/05_presence.md).

**Q: Can I have nested hives (hub-to-hub communication)?**

A: Yes! One hub can connect to another as a "slave mind". See [Nested Hives](/HiveMind-community-docs/docs/15_nested.md).

---

## Troubleshooting

**Q: Hub won't start / connection refused**

A:
1. Check port availability: `lsof -i :5678`
2. Verify configuration: `cat ~/.config/hivemind-core/server.json`
3. Check logs: `journalctl -u hivemind-core` (if using systemd)
4. Ensure all required packages installed: `uv pip install hivemind-core[full]`

**Q: Satellite can't connect to hub**

A:
1. Verify hub is running and reachable: `ping <hub-ip>`
2. Check credentials in identity file: `~/.config/hivemind-core/identity.json`
3. Verify firewall allows port 5678 (WebSocket) or 5679 (HTTP)
4. Test password handshake: `hivemind-client check-password`

**Q: Audio is choppy or delayed**

A:
- Reduce network latency (use 5GHz WiFi, wired Ethernet if possible)
- Switch to local WakeWord detection (Voice Satellite/Relay rather than Mic Satellite)
- Increase audio buffer size in satellite config
- Check hub CPU usage; may need more powerful hardware

---

## Development

**Q: How do I write a custom satellite?**

A: Use the [Client Libraries](/HiveMind-community-docs/docs/11_devs.md). See `HiveMessageBusClient` Python API or `HiveMindJs` for JavaScript/browser.

**Q: How do I test my changes?**

A: Use the [Testing Guide](/HiveMind-community-docs/docs/testing.md) with the `hivemind-test-harness` package.

**Q: Where's the full API documentation?**

A: Check the individual package READMEs in the [HiveMind GitHub org](https://github.com/JarbasHiveMind).

**Q: Where is the official documentation hosted?**

A: It is hosted on GitHub Pages at https://jarbashivemind.github.io/HiveMind-community-docs/.

**Q: How can I contribute to the documentation?**

A: You can contribute by submitting pull requests to the repository. All documentation is written in Markdown and managed by MkDocs.

**Q: How do I build the documentation locally?**

A: Install `mkdocs` and dependencies:
```bash
pip install mkdocs mkdocs-material mkdocs-readthedocs-dark
mkdocs serve
```

Then navigate to `http://localhost:8000`.
