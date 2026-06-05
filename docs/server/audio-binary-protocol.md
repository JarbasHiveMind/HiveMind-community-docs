# Audio Binary Protocol

`hivemind-audio-binary-protocol` is a binary data handler plugin for `hivemind-core`. It enables the hub to receive audio streams from satellites, run wakeword detection, STT, and TTS synthesis, and return synthesized audio to the satellite.

Without this plugin, `hivemind-core` cannot process audio. It is required for mic-satellite and voice-relay.

## Install

```bash
pip install hivemind-audio-binary-protocol
```

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

## Audio flow

```
[mic-satellite]                     [HiveMind Hub]
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

## Binary payload types

| Value | Name | Description |
|---|---|---|
| 1 | `RAW_AUDIO` | Continuous microphone stream (VAD-gated) |
| 4 | `STT_AUDIO_TRANSCRIBE` | Full audio sentence — return transcript only |
| 5 | `STT_AUDIO_HANDLE` | Full audio sentence — transcribe and handle intent |
| 6 | `TTS_AUDIO` | Synthesized speech audio (hub → satellite) |

## The service model

STT and TTS running on the hub via this plugin are authenticated by the same access-key mechanism as all other HiveMind messages. A satellite cannot choose the engine or voice — the hub operator configures them centrally. This is intentional: the hive owns its speech services, just as it owns its skills.

## Security note

Running STT/TTS through the audio binary protocol requires the satellite to have valid HiveMind credentials. There is no open audio endpoint — access is gated by the client's access key and permission settings.
