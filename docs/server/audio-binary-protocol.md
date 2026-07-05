# Audio Binary Protocol

A thin satellite — a bare microcontroller, a mic satellite, a voice relay — can capture
sound but can't make sense of it. So it ships the raw audio to the server and asks the
server to listen *for* it. This plugin is the ear on the server side: install it and
hivemind-core can take an incoming audio stream, spot the wake word, transcribe the
speech, run the skill, synthesize the reply, and send the finished audio back for the
device to play. Without it, hivemind-core has no way to touch audio at all — which is why
the mic satellite and voice relay simply won't work until it's in place.

!!! abstract "In a nutshell"
    - A `hivemind.binary.protocol` plugin that gives hivemind-core ears — required for the mic satellite and voice relay.
    - The server owns the wakeword/STT/TTS/VAD engines centrally, so a satellite can't choose the engine or the voice.
    - There's no open audio endpoint: a client still needs a valid access key and the right permissions before a single byte is processed.

---

## Install

```bash
pip install hivemind-audio-binary-protocol
```

---

## Configuration

In `~/.config/hivemind-core/server.json`, set the `binary_protocol` section:

```json
{
  "binary_protocol": {
    "module": "hivemind-audio-binary-protocol-plugin",
    "hivemind-audio-binary-protocol-plugin": {
      "stt": {
        "module": "ovos-stt-plugin-server",
        "ovos-stt-plugin-server": {
          "url": "https://stt.openvoiceos.org/stt"
        }
      },
      "tts": {
        "module": "ovos-tts-plugin-server",
        "ovos-tts-plugin-server": {
          "host": "https://tts.openvoiceos.org"
        }
      },
      "vad": {
        "module": "ovos-vad-plugin-silero"
      },
      "wake_word": "hey_mycroft",
      "hotwords": {
        "hey_mycroft": {
          "module": "ovos-ww-plugin-precise-lite",
          "model": "https://github.com/OpenVoiceOS/precise-lite-models/raw/master/wakewords/en/hey_mycroft.tflite"
        }
      }
    }
  }
}
```

The plugin and its wakeword/STT/TTS/VAD plugins must all be installed:

```bash
pip install ovos-vad-plugin-silero
pip install ovos-ww-plugin-precise-lite
pip install ovos-stt-plugin-server     # or any other STT plugin
pip install ovos-tts-plugin-server     # or any other TTS plugin
```

---

## Audio flow

```
[mic-satellite]                     [hivemind-core]
  Mic + VAD                         AudioBinaryProtocol
     │                                    │
     │── BINARY (RAW_AUDIO) ─────────────→│
     │                               WakeWord detection
     │                                    │ (after wakeword)
     │                               STT transcription
     │                                    │
     │                               OVOS IntentService
     │                                    │
     │                               TTS synthesis
     │                                    │
     │←── BINARY (TTS_AUDIO) ────────────┤
  Playback
```

---

## Binary payload types

| Value | Name | Description |
|---|---|---|
| 1 | `RAW_AUDIO` | Continuous microphone stream (VAD-gated) |
| 4 | `STT_AUDIO_TRANSCRIBE` | Full audio sentence — return transcript only |
| 5 | `STT_AUDIO_HANDLE` | Full audio sentence — transcribe and handle intent |
| 6 | `TTS_AUDIO` | Synthesized speech audio (server → satellite) |

---

## The service model

STT and TTS running on `hivemind-core` via this plugin are authenticated by the same access-key mechanism as all other HiveMind messages. A satellite cannot choose the engine or voice — the server operator configures them centrally. This is intentional: the hive owns its speech services, just as it owns its skills.

---

## Security note

Running STT/TTS through the audio binary protocol requires the satellite to have valid HiveMind credentials. There is no open audio endpoint — access is gated by the client's access key and permission settings.

---

## Source

Validated against the HiveMind source:

- [`setup.py`](https://github.com/JarbasHiveMind/hivemind-audio-binary-protocol/blob/HEAD/setup.py) — the `hivemind-audio-binary-protocol-plugin` entry point in the `hivemind.binary.protocol` group
- [`hivemind_audio_binary_protocol/protocol.py`](https://github.com/JarbasHiveMind/hivemind-audio-binary-protocol/blob/HEAD/hivemind_audio_binary_protocol/protocol.py) — the binary payload types and the wakeword/STT/TTS/VAD audio flow
