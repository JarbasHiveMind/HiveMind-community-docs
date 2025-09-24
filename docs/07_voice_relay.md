# HiveMind Voice Relay (voice-relay)

The **Voice Relay** runs a lightweight OpenVoiceOS stack locally. Only the **microphone, VAD, and wake word** are processed on-device. **STT and TTS are offloaded to HiveMind Core** using the **binary plugin**, which securely streams audio and handles processing on the hub.

> ⚠️ Requires HiveMind Core with the binary plugin loaded to provide STT/TTS services.

Built on [ovos-simple-listener](https://github.com/TigreGotico/ovos-simple-listener) and [ovos-audio](https://github.com/OpenVoiceOS/ovos-audio/).

---

## Install

```bash
pip install HiveMind-voice-relay
```

---

## Usage

```bash
Usage: hivemind-voice-relay [OPTIONS]

  connect to HiveMind

Options:
  --host TEXT      HiveMind host
  --key TEXT       Access Key
  --password TEXT  Password for key derivation
  --port INTEGER   HiveMind port number
  --selfsigned     Accept self signed certificates
  --help           Show this message and exit.
```

---

## Configuration

The Voice Relay uses the **standard OpenVoiceOS configuration** at:

```
~/.config/mycroft/mycroft.conf
```

All plugin settings (microphone, VAD, wake word, playback, G2P, PHAL, etc.) are managed through this configuration.

See the [OVOS plugin documentation](https://openvoiceos.github.io/ovos-technical-manual/) for details.

---

## Key Features

* ✅ Local processing: microphone, VAD, wake word
* 🌐 Hub processing: STT and TTS handled via HiveMind **binary plugin**
* 🔒 Secure streaming: only audio after wake-word is sent
* 📦 Compatible with existing OVOS STT/TTS plugins on the hub
* ⚡ Lightweight: suitable for mid-range devices

---

## Notes

* Continuous listening, hybrid listening, and some advanced transformer plugins are **not supported locally**.
* Ideal for devices that can detect wake word but lack resources for full STT/TTS.
* All HiveMind communication is encrypted and secure.

