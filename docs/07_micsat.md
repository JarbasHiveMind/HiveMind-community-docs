# HiveMind Microphone Satellite (mic-satellite)

The Microphone Satellite is an ultra-lightweight OpenVoiceOS device. It runs only the microphone and VAD locally, and streams audio to HiveMind Core for all further processing, including wake word, STT, and TTS, through the binary plugin.

> Requires HiveMind Core with a binary plugin loaded, to provide STT/TTS and wake word detection.

Built on [ovos-simple-listener](https://github.com/TigreGotico/ovos-simple-listener) and [ovos-audio](https://github.com/OpenVoiceOS/ovos-audio/).

---

## Install

```bash
pip install hivemind-mic-satellite
```

---

## Usage

```bash
Usage: hivemind-mic-satellite [OPTIONS]

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

The Microphone Satellite uses the standard OpenVoiceOS configuration at:

```
~/.config/mycroft/mycroft.conf
```

This file manages all plugin settings: microphone, VAD, PHAL, G2P, media playback, and more.

See the [OVOS plugin documentation](https://openvoiceos.github.io/ovos-technical-manual/) for details.

---

## Key features

- Local processing: microphone and VAD only.
- Hub processing: wake word, STT, TTS, and transformers through the HiveMind binary plugin.
- Secure streaming of audio to HiveMind Core.
- Ultra-lightweight, suited to IoT and resource-constrained devices.

---

## Notes

- No wake word detection, STT, or TTS runs locally.
- This satellite fits very low-resource devices where all heavy processing offloads to the hub.
- All HiveMind communication is encrypted and secure.

---
[← Voice Relay](07_voice_relay.md) · [Home](index.md) · [OVOS Pipeline →](pipeline.md)
