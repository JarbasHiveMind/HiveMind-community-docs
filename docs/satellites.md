
# HiveMind Satellite Overview

HiveMind supports multiple types of **satellite devices** that connect to the hub. Satellites can be configured based on **device capabilities**, **privacy requirements**, and **network bandwidth**.

All satellites use the **standard OpenVoiceOS `mycroft.conf`** for plugin configuration. Refer to the [OVOS technical manual](https://openvoiceos.github.io/ovos-technical-manual/) for plugin specific information.

---

## Satellite Options

### 1. **Voice Satellite (voice-sat)**

* **Audio handling:** Fully local (microphone, VAD, wake word, STT, TTS)
* **HiveMind role:** Sends only **text messages** to the hub
* **Best for:** Privacy-conscious devices, offline operation, low latency

---

### 2. **Voice Relay (voice-relay)**

* **Audio handling:** Local microphone, VAD, wake word
* **HiveMind role:** Handles **STT and TTS** via the **binary plugin**; streams only triggered audio
* **Best for:** Medium-powered devices that can run wake word detection but not STT/TTS

---

### 3. **Microphone Satellite (mic-satellite)**

* **Audio handling:** Only microphone and VAD locally
* **HiveMind role:** All processing runs on the hub (wake word, STT, TTS) via the **binary plugin**; streams continuous or VAD-triggered audio
* **Best for:** Low-resource devices, IoT microphones, centralized processing


---

## Choosing the Right Satellite

| Satellite     | Local Audio      | Hub Processing   | Typical Use Case                               |
| ------------- |------------------| ---------------- | ---------------------------------------------- |
| voice-sat     | Full audio stack | Text only        | Privacy-first, offline-capable devices         |
| voice-relay   | Wake + VAD       | STT & TTS        | Mid-range devices, minimize bandwidth          |
| mic-satellite | VAD              | Wake + STT + TTS | Tiny IoT devices, fully centralized processing |

---

## Security & compatibility

* **All audio and binary payloads travel over HiveMind’s secure transport.** The binary plugin uses the same encrypted transport as normal BUS messages, binary streaming is not a separate protocol.
* The binary plugin on the hub **loads OpenVoiceOS STT and TTS plugins**, making HiveMind compatible with the existing OVOS plugin ecosystem (hundreds of available plugins).
* Use the standard `mycroft.conf` to configure plugins; no special satellite-specific configuration format is required.

---

## Troubleshooting & tips

* **High latency**: check network bandwidth; prefer `voice-sat` if latency must be minimal.
* **Bandwidth concerns**: prefer `voice-relay` (stream only post-wake) instead of `mic-satellite`.
* **Certificate errors**: use `--selfsigned` for testing, but deploy proper TLS certificates for production.
* **Server setup**: ensure your HiveMind core has the binary plugin enabled and has compatible OVOS STT/TTS plugins installed.
