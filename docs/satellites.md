# HiveMind Satellite Overview

HiveMind supports several types of satellite devices that connect to the hub. Choose a satellite based on device capabilities, privacy requirements, and network bandwidth.

All satellites use the standard OpenVoiceOS `mycroft.conf` for plugin configuration. See the [OVOS technical manual](https://openvoiceos.github.io/ovos-technical-manual/) for plugin-specific information.

---

## Satellite options

### 1. Voice Satellite (voice-sat)

- **Audio handling**: fully local (microphone, VAD, wake word, STT, TTS).
- **HiveMind role**: sends only text messages to the hub.
- **Best for**: privacy-conscious devices, offline operation, low latency.

### 2. Voice Relay (voice-relay)

- **Audio handling**: local microphone, VAD, wake word.
- **HiveMind role**: handles STT and TTS through the binary plugin; streams only triggered audio.
- **Best for**: medium-powered devices that can run wake word detection but not STT/TTS.

### 3. Microphone Satellite (mic-satellite)

- **Audio handling**: only microphone and VAD locally.
- **HiveMind role**: all processing runs on the hub (wake word, STT, TTS) through the binary plugin; streams continuous or VAD-triggered audio.
- **Best for**: low-resource devices, IoT microphones, centralized processing.

---

## Choosing the right satellite

| Satellite     | Local Audio      | Hub Processing   | Typical Use Case                               |
| ------------- |------------------| ---------------- | ---------------------------------------------- |
| voice-sat     | Full audio stack | Text only        | Privacy-first, offline-capable devices         |
| voice-relay   | Wake + VAD       | STT & TTS        | Mid-range devices, minimize bandwidth          |
| mic-satellite | VAD              | Wake + STT + TTS | Tiny IoT devices, fully centralized processing |

---

## Security and compatibility

- All audio and binary payloads travel over HiveMind's secure transport. The binary plugin uses the same encrypted transport as normal BUS messages; binary streaming is not a separate protocol.
- The binary plugin on the hub loads OpenVoiceOS STT and TTS plugins, so HiveMind stays compatible with the existing OVOS plugin ecosystem.
- Use the standard `mycroft.conf` to configure plugins; no separate satellite-specific configuration format exists.

---

## Troubleshooting and tips

- **High latency**: check network bandwidth; prefer `voice-sat` if latency must be minimal.
- **Bandwidth concerns**: prefer `voice-relay` (stream only after wake) instead of `mic-satellite`.
- **Certificate errors**: use `--selfsigned` for testing, but deploy proper TLS certificates for production.
- **Server setup**: make sure your HiveMind core has the binary plugin enabled and compatible OVOS STT/TTS plugins installed.

---
[← Database](db_plugins.md) · [Home](index.md) · [Voice Satellite →](07_voicesat.md)
