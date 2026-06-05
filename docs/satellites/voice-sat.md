# Voice Satellite

The full-stack OVOS voice satellite. Microphone, VAD, wakeword, STT, and TTS all run on this device. Only the transcribed text utterance is sent to the hub. Responses arrive as text and are spoken locally.

**What runs locally**: Microphone + VAD + Wakeword + STT + TTS

**What the hub provides**: Skills, intents, reasoning

## When to use it

- Devices with sufficient CPU/GPU (laptop, desktop, Raspberry Pi 4B or better)
- Maximum audio privacy — audio never leaves the device
- Low-latency interaction (no round-trip for audio)
- Offline operation after initial model downloads

## Install

```bash
pip install HiveMind-voice-sat

# Linux (ALSA / SoundDevice microphone support)
pip install HiveMind-voice-sat[linux]

# macOS
pip install HiveMind-voice-sat[mac]
```

## Quickstart

**1. On the hub** — register a client:

```bash
hivemind-core add-client --name my-voice-sat
```

**2. On the satellite** — run it:

```bash
hivemind-voice-sat --host <hub_host> --key <access_key> --password <password>
```

Say your wakeword. The satellite transcribes locally and sends the utterance text to the hub.

## CLI reference

```
Usage: hivemind-voice-sat [OPTIONS]

Options:
  --host TEXT      HiveMind host (ws:// or wss://)
  --key TEXT       Access key
  --password TEXT  Password for key derivation
  --port INTEGER   HiveMind port number (default: 5678)
  --selfsigned     Accept self-signed TLS certificates
  --siteid TEXT    Location identifier for message context
```

All flags fall back to the identity file written by `hivemind-client set-identity`.

## Configuration

voice-sat reads `~/.config/mycroft/mycroft.conf` (standard OVOS configuration). Override STT, TTS, VAD, and wakeword plugins there.

Example:

```json
{
  "listener": {
    "VAD": {
      "module": "ovos-vad-plugin-silero"
    },
    "wake_word": "hey_mycroft",
    "hey_mycroft": {
      "module": "ovos-ww-plugin-vosk"
    }
  },
  "stt": {
    "module": "ovos-stt-plugin-server",
    "ovos-stt-plugin-server": { "url": "https://stt.openvoiceos.org/stt" }
  },
  "tts": {
    "module": "ovos-tts-plugin-server",
    "ovos-tts-plugin-server": { "host": "https://tts.openvoiceos.org" }
  }
}
```

All plugin slots are swappable via [ovos-plugin-manager](https://github.com/OpenVoiceOS/ovos-plugin-manager).

| Plugin type | Config key | Required |
|---|---|---|
| Microphone | `listener.microphone.module` | Yes |
| VAD | `listener.VAD.module` | Yes |
| Wakeword | `listener.wake_word` | Yes (can skip with continuous listening) |
| STT | `stt.module` | Yes |
| TTS | `tts.module` | Yes |
| G2P | `g2p.module` | No |
| Media playback | various | No |
| Audio/Dialog transformers | various | No |
| PHAL | `PHAL.ovos-phal-...` | No |

## How it differs from other satellites

Unlike mic-satellite and voice-relay, voice-sat sends **text utterances** to the hub — no audio leaves the device. The hub only needs to run skills and return text responses. It does not need `hivemind-audio-binary-protocol`.

You also choose your own STT and TTS plugins, independent of any hub configuration.
