# HiveMind Voice Relay (voice-relay)

The Voice Relay runs a lightweight OpenVoiceOS stack locally. Only the microphone, VAD, and wake word run on-device. STT and TTS run on HiveMind Core through the binary plugin, which securely streams audio and processes it on the hub.

> Requires HiveMind Core with the binary plugin loaded, to provide STT/TTS services.

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

The Voice Relay uses the standard OpenVoiceOS configuration at:

```
~/.config/mycroft/mycroft.conf
```

This file manages all plugin settings: microphone, VAD, wake word, playback, G2P, PHAL, and more.

See the [OVOS plugin documentation](https://openvoiceos.github.io/ovos-technical-manual/) for details.

---

## Key features

- Local processing: microphone, VAD, wake word.
- Hub processing: STT and TTS through the HiveMind binary plugin.
- Secure streaming: only audio after the wake word goes to the hub.
- Compatible with existing OVOS STT/TTS plugins on the hub.
- Lightweight, suited to mid-range devices.

---

## Notes

- Continuous listening, hybrid listening, and some advanced transformer plugins are not supported locally.
- This satellite fits devices that can detect a wake word but lack resources for full STT/TTS.
- All HiveMind communication is encrypted and secure.

---
[← Voice Satellite](07_voicesat.md) · [Home](index.md) · [Mic Satellite →](07_micsat.md)
