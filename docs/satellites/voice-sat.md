# Voice Satellite

**The voice satellite is the full-stack OVOS satellite: microphone, VAD, wakeword, and the STT/TTS plugins all run on the device.** Only the transcribed text utterance is sent to hivemind-core — never audio. Responses arrive as text and are spoken locally.

!!! abstract "In a nutshell"
    - **Runs locally**: microphone, VAD, wakeword, plus STT and TTS.
    - **hivemind-core provides**: skills, intents, and reasoning — no `hivemind-audio-binary-protocol` needed.
    - Sends only text utterances, so audio never reaches hivemind-core and STT/TTS are chosen independently of any hivemind-core config.
    - The shipped default plugins (`ovos-stt-plugin-server` / `ovos-tts-plugin-server`) are remote HTTP services; swap in local plugins for fully on-device, offline operation.

> **Note**: STT and TTS are chosen on the satellite, but the *shipped defaults*
> (`ovos-stt-plugin-server` / `ovos-tts-plugin-server`) are remote HTTP services
> pointed at `*.openvoiceos.org`. Audio stays off hivemind-core, but with the defaults it
> still leaves the device for transcription. For true on-device/offline operation,
> swap in local STT/TTS plugins (see [Configuration](#configuration)).

**What runs locally**: Microphone + VAD + Wakeword (+ STT + TTS if local plugins are configured)

**What hivemind-core provides**: Skills, intents, reasoning

---

## When to use it

- Devices with sufficient CPU/GPU (laptop, desktop, Raspberry Pi 4B or better)
- Maximum audio privacy — **only if** you swap in local STT/TTS plugins. The shipped
  defaults (`ovos-stt-plugin-server` / `ovos-tts-plugin-server`) send audio to a
  remote speech server (configured against `*.openvoiceos.org`).
- Low-latency interaction (no round-trip for audio)
- Fully offline operation after initial model downloads — again, only with local
  STT/TTS plugins in place of the remote-server defaults

---

## Install

```bash
pip install HiveMind-voice-sat

# Linux (ALSA / SoundDevice microphone support)
pip install HiveMind-voice-sat[linux]

# macOS
pip install HiveMind-voice-sat[mac]
```

---

## Quickstart

**1. On hivemind-core** — register a client:

```bash
hivemind-core add-client --name my-voice-sat
```

**2. On the satellite** — run it:

```bash
hivemind-voice-sat --host <server_host> --key <access_key> --password <password>
```

Say your wakeword. The satellite transcribes locally and sends the utterance text to hivemind-core.

!!! tip "Pairing by sound (no `--password`)"
    If you start voice-sat **without** a `--password` (and none is stored in the
    identity file), it does not error out. Instead it launches
    [hivemind-ggwave](https://github.com/JarbasHiveMind/hivemind-ggwave)'s
    `GGWaveSlave` and **listens for the password to arrive over sound** — a short
    audio "chirp" played near the device. Once it receives and stores the credentials,
    it connects automatically. This lets you pair a headless device by playing the
    pairing tone at it instead of typing a password.

---

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

---

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

---

## How it differs from other satellites

Unlike mic-satellite and voice-relay, voice-sat sends **text utterances** to hivemind-core — no audio reaches hivemind-core. hivemind-core only needs to run skills and return text responses. It does not need `hivemind-audio-binary-protocol`. (With the default remote STT/TTS plugins, audio is still sent to the configured speech server for transcription; use local plugins to keep all audio on-device.)

You also choose your own STT and TTS plugins, independent of any hivemind-core configuration.

---

## Next

Give hivemind-core something to answer with: set up an [OVOS Skills backend](../server/ovos-server.md).

---

## Source

Validated against the HiveMind source:

- [`hivemind_voice_satellite/__main__.py`](https://github.com/JarbasHiveMind/HiveMind-voice-sat/blob/HEAD/hivemind_voice_satellite/__main__.py) — CLI flags (`--host --key --password --port --selfsigned --siteid`) and the ggwave audio-password pairing path
- [`hivemind_voice_satellite/service.py`](https://github.com/JarbasHiveMind/HiveMind-voice-sat/blob/HEAD/hivemind_voice_satellite/service.py) — the local STT/TTS voice client
- [`requirements/requirements.txt`](https://github.com/JarbasHiveMind/HiveMind-voice-sat/blob/HEAD/requirements/requirements.txt) — shipped default plugins (`ovos-stt-plugin-server` / `ovos-tts-plugin-server`, remote HTTP) and `hivemind-ggwave`
