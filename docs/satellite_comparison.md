# Choosing a Satellite

HiveMind satellites form a spectrum based on where processing happens. Choose the right satellite for your use case:

## Quick Comparison

| Satellite | Local Processing | Hub Processing | Use Case | Resources |
|---|---|---|---|---|
| **HiveMind-cli** | None (text input only) | All | Text-only clients, bots, testing | Minimal |
| **Mic Satellite** | Microphone + VAD | STT, TTS, wakeword, skills | Lightweight voice devices | Low |
| **Voice Relay** | Microphone + VAD + Wakeword | STT, TTS, skills | Voice devices with wakeword | Medium |
| **Voice Satellite** | Microphone + VAD + Wakeword + STT + TTS | Skills, intents | Full local voice processing | High |
| **WebSpeech Browser** | Microphone + VAD (browser-based) | STT, TTS, skills | Web browser clients | Browser-dependent |

## Detailed Overview

### HiveMind-cli

A text-only command-line client with no audio processing.

**What runs locally**: Nothing — just a CLI interface.

**What the hub provides**: All processing (STT, TTS, skills, intents).

**Best for**:
- Testing and debugging
- Scripting and automation
- Server-side bots (no audio hardware needed)
- IoT devices with only network connectivity

**Installation**:
```bash
pip install hivemind-cli
```

**Usage**:
```bash
hivemind-client --host 192.168.1.10 --key <key> --password <password>
```

**Pros**: Simplest to deploy, minimal dependencies, great for testing.

**Cons**: No local voice input or output.

---

### HiveMind Microphone Satellite

A lightweight satellite that streams raw microphone audio to the hub.

**What runs locally**: Microphone capture + Voice Activity Detection (VAD).

**What the hub provides**: STT, TTS, wakeword detection, skills, intents.

**Best for**:
- Embedded devices (Raspberry Pi Zero, Beaglebone)
- Devices with very limited processing power
- Scenarios where you want minimal local configuration
- Microphone-only devices (no speaker)

**Installation**:
```bash
pip install hivemind-mic-satellite
```

**Configuration**: Uses OVOS configuration (`~/.config/mycroft/mycroft.conf`). Supports microphone and VAD plugins.

**Plugins**:
- Microphone (required)
- VAD (required)
- PHAL (optional, platform support)
- G2P (optional, viseme support)
- Media playback (optional)

**Pros**: Ultra-lightweight, minimal CPU/RAM, simple setup.

**Cons**: Requires hub to provide STT/TTS, no local wakeword detection, continuous network bandwidth for audio streaming.

---

### HiveMind Voice Relay

A lightweight satellite with local wakeword detection.

**What runs locally**: Microphone + VAD + Wakeword detection.

**What the hub provides**: STT, TTS, skills, intents.

**Best for**:
- Budget voice devices (Raspberry Pi 3/4)
- Devices that need wakeword-based interaction
- Users who want to save bandwidth vs. mic-satellite
- Environments where internet latency is acceptable for STT/TTS

**Installation**:
```bash
pip install HiveMind-voice-relay
```

**Configuration**: Uses OVOS configuration (`~/.config/mycroft/mycroft.conf`). Built on `ovos-simple-listener`.

**Plugins**:
- Microphone (required)
- VAD (required)
- Wakeword (required)
- G2P (optional)
- Media playback (optional)
- PHAL (optional)

**Pros**: Reduced bandwidth vs. mic-satellite, local wakeword saves latency, good balance of features and resources.

**Cons**: No local STT/TTS (still depends on hub), moderate CPU/RAM requirements.

---

### HiveMind Voice Satellite

Full-featured satellite with complete local voice processing.

**What runs locally**: Microphone + VAD + Wakeword + STT + TTS.

**What the hub provides**: Skills, intents, reasoning.

**Best for**:
- Powerful devices (laptop, desktop, Raspberry Pi 4B+)
- Users who want true offline voice capability
- Low-latency voice interaction
- Maximum privacy (audio never leaves device)

**Installation**:
```bash
sudo apt-get install -y libpulse-dev libasound2-dev
pip install HiveMind-voice-sat
```

**Configuration**: Uses OVOS configuration (`~/.config/mycroft/mycroft.conf`). Built on `ovos-dinkum-listener`.

**Plugins**:
- Microphone (required)
- VAD (required)
- Wakeword (required, can skip with continuous listening)
- STT (required)
- TTS (required)
- G2P (optional)
- Media playback (optional)
- Audio/Dialog transformers (optional)
- PHAL (optional)

**Pros**: Complete offline voice, best latency, maximum privacy, all OVOS plugins supported.

**Cons**: Higher resource requirements, needs STT/TTS model downloads.

---

### WebSpeech Browser Satellite

A JavaScript-based satellite for web browsers.

**What runs locally**: Microphone capture + VAD (in-browser).

**What the hub provides**: STT, TTS, skills, intents.

**Best for**:
- Web applications and chat interfaces
- Users already in a browser
- Server-side hub on a local network or VPS
- Temporary client access

**Installation**: Include the HiveMind.js library in your HTML.

**Configuration**: Configured via JavaScript API.

**Pros**: No installation required, works in any modern browser, easy integration.

**Cons**: Browser-dependent, requires JavaScript support, may have browser-specific VAD limitations.

---

## Decision Tree

1. **Do you need voice input?**
   - No → Use **HiveMind-cli**
   - Yes → Continue

2. **Can your device run local STT/TTS models?**
   - No → Continue
   - Yes → Use **HiveMind-Voice-Satellite**

3. **Do you need wakeword detection?**
   - No → Use **HiveMind-Mic-Satellite**
   - Yes → Use **HiveMind-Voice-Relay**

4. **Is this a web browser?**
   - Yes → Use **WebSpeech Browser Satellite**
   - No → Follow the steps above

---

## Hub Requirements by Satellite

| Satellite | Hub Must Provide | Recommended Hub Setup |
|---|---|---|
| HiveMind-cli | Skills processing | `hivemind-core` + OVOS or `hivemind-persona` |
| Mic Satellite | STT, TTS, wakeword | `hivemind-core` + `hivemind-audio-binary-protocol` |
| Voice Relay | STT, TTS | `hivemind-core` + `hivemind-audio-binary-protocol` OR `hivemind-core` + OVOS/ovos-audio |
| Voice Satellite | Skills, intents | `hivemind-core` + OVOS or `hivemind-persona` |
| WebSpeech Browser | STT, TTS, skills | `hivemind-core` + `hivemind-audio-binary-protocol` or any hub |

See [Quick Start](01_quickstart.md) to set up a hub and connect your first satellite.
