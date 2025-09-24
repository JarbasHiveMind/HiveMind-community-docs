# HiveMind Microphone Satellite (mic-satellite)

The **Microphone Satellite** is an ultra-lightweight OpenVoiceOS device. It runs only the **microphone and VAD locally**, streaming audio to HiveMind Core for all further processing, including **wake word, STT, and TTS**, via the **binary plugin**.

> ⚠️ Requires HiveMind Core with a binary plugin loaded to provide STT/TTS and wake word detection.

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

The Microphone Satellite uses the **standard OpenVoiceOS configuration** at:

```
~/.config/mycroft/mycroft.conf
```

All plugin settings (microphone, VAD, PHAL, G2P, media playback, etc.) are managed through this configuration.

See the [OVOS plugin documentation](https://openvoiceos.github.io/ovos-technical-manual/) for details.

---

## Key Features

* ✅ Local processing: microphone and VAD only
* 🌐 Hub processing: wake word, STT, TTS, transformers handled via HiveMind **binary plugin**
* 🔒 Secure streaming of audio to HiveMind Core
* ⚡ Ultra-lightweight: suitable for IoT and resource-constrained devices

---

## Notes

* No wake word detection, STT, or TTS runs locally.
* Ideal for very low-resource devices where all heavy processing is offloaded to the hub.
* All HiveMind communication is encrypted and secure.

