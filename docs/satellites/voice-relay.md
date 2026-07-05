# Voice Relay

**Voice-relay runs microphone, VAD, and wakeword detection locally, then forwards audio to hivemind-core only after the wakeword triggers.** STT and TTS synthesis run on hivemind-core.

!!! abstract "In a nutshell"
    - **Runs locally**: microphone + VAD + wakeword.
    - **hivemind-core provides**: STT and TTS — requires `hivemind-audio-binary-protocol` on hivemind-core.
    - Audio leaves the device only after wakeword activation, saving bandwidth and improving privacy over the [mic satellite](mic-satellite.md).
    - STT/TTS are centrally governed by the hivemind-core operator — the "HiveMind as a service" model — so a relay cannot pick its own speech engines.

---

## When to use it

- Devices with enough CPU for a wakeword engine (Raspberry Pi 3/4, similar)
- Deployments where you want STT/TTS to be centrally governed (HiveMind as a service)
- Scenarios where audio should not leave the device until activation (latency + privacy)
- Multi-user or at-scale service deployments

---

## The service model

Voice-relay's architecture has a specific meaning: STT and TTS run inside the hive (the `hivemind-audio-binary-protocol` plugin on `hivemind-core`) and are gated by the same access-key authentication as all other HiveMind messages. The hivemind-core operator chooses the STT engine, TTS engine, and voice. A relay satellite cannot override them.

This is the difference from voice-sat: a voice-sat can point at any STT/TTS plugin it likes. A relay cannot.

---

## Install

```bash
pip install HiveMind-voice-relay
```

---

## Server requirements

hivemind-core must have `hivemind-audio-binary-protocol` installed and configured. Alternatively, run `hivemind-core` together with `ovos-audio` and `ovos-dinkum-listener` to provide equivalent capabilities.

See [Audio Binary Protocol](../server/audio-binary-protocol.md).

---

## Quickstart

> **Pre-flight:** wakeword runs locally, but after activation this satellite sends audio to hivemind-core, so **hivemind-core MUST have the [Audio Binary Protocol](../server/audio-binary-protocol.md) configured** — it provides STT and TTS. Without it the satellite connects but its audio is silently dropped, with no error.

**1. On hivemind-core** — register a client:

```bash
hivemind-core add-client --name my-voice-relay
```

**2. On the satellite** — write the identity file:

```bash
hivemind-client set-identity \
  --key <access_key> \
  --password <password> \
  --host <server_host>
```

**3. Run:**

```bash
hivemind-voice-relay
```

**4. Say your wake word.** Default is `hey mycroft` (configured in `~/.config/mycroft/mycroft.conf`).

---

## CLI flags

```
Usage: hivemind-voice-relay [OPTIONS]

Options:
  --host TEXT      HiveMind host (ws:// or wss://)
  --key TEXT       Access key
  --password TEXT  Password for key derivation
  --port INTEGER   HiveMind port number (default: 5678)
  --selfsigned     Accept self-signed TLS certificates
  --siteid TEXT    Location identifier for message context
```

---

## Configuration

voice-relay reads `~/.config/mycroft/mycroft.conf`.

| Plugin type | Config key | Required |
|---|---|---|
| Microphone | `listener.microphone.module` | Yes |
| VAD | `listener.VAD.module` | Yes |
| Wakeword | `listener.wake_word` | Yes |
| G2P | `g2p.module` | No |
| Media playback | various | No |
| PHAL | `PHAL.ovos-phal-...` | No |

---

## How audio flows

```
[Microphone] → [VAD] → [Wakeword detector]
                              ↓ (after activation)
                    audio chunk → [hivemind-core]
                                       ↓
                             [hivemind-audio-binary-protocol]
                                       ↓
                               STT → IntentService
                                       ↓
                               TTS synthesis → audio
                                       ↓
                              [Satellite plays TTS audio]
```

After the wakeword triggers, audio is sent as base64-encoded audio via `recognizer_loop:b64_transcribe`. TTS audio comes back via `speak:b64_audio`.

---

## Next

Set up the server side: [Audio Binary Protocol](../server/audio-binary-protocol.md).

---

## Source

Validated against the HiveMind source:

- [`hivemind_voice_relay/service.py`](https://github.com/JarbasHiveMind/HiveMind-voice-relay/blob/HEAD/hivemind_voice_relay/service.py) — base64-over-bus transport: STT via `recognizer_loop:b64_transcribe`, TTS via `speak:b64_audio`
- [`hivemind_voice_relay/__main__.py`](https://github.com/JarbasHiveMind/HiveMind-voice-relay/blob/HEAD/hivemind_voice_relay/__main__.py) — CLI flags (`--host --key --password --port --selfsigned --siteid`)
- [`docs/architecture.md`](https://github.com/JarbasHiveMind/HiveMind-voice-relay/blob/HEAD/docs/architecture.md) — local mic + VAD + wakeword, server-owned STT/TTS
