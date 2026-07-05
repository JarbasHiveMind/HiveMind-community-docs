# Microphone Satellite

Take an old Raspberry Pi Zero, plug in a microphone, and let it do almost nothing but
listen. The mic satellite runs just enough to notice *when* someone is speaking — a
microphone and voice-activity detection — and streams that audio straight to
hivemind-core. It doesn't even wait for a wake word; the server handles that too, along
with the transcription, the skills, and the spoken reply. That makes it the simplest
software satellite to run and the least demanding on the device. The trade-off is candid:
because it forwards every scrap of detected speech, it's chatty on the network and leans
hard on the server, so it's happiest on a small, fast home LAN.

!!! abstract "In a nutshell"
    - **On the device:** a microphone and VAD. That's it.
    - **On hivemind-core:** wakeword, STT, intents, and TTS — which means the server needs `hivemind-audio-binary-protocol` installed.
    - It streams every VAD-gated segment of speech as binary `RAW_AUDIO`, so it's bandwidth-heavy — a good fit for a handful of devices on a homelab LAN.
    - Serving many users or running at scale? Reach for [voice-relay](voice-relay.md) instead — it holds audio back until a local wake word fires.

---

## When to use it

- Devices with very limited CPU/RAM (Raspberry Pi Zero, recycled phones)
- Scenarios where you want zero local model downloads
- Personal homelab with a handful of devices on a fast LAN

---

## When not to use it

mic-satellite streams every detected voice segment to hivemind-core continuously (VAD-gated but not wakeword-gated). This is bandwidth-intensive and puts the full STT load on the server for all speech activity. It is appropriate for a small homelab. For a deployment serving multiple users, prefer [voice-relay](voice-relay.md) — local wakeword means audio only leaves the device after activation.

---

## Install

```bash
pip install hivemind-mic-satellite
```

Requires Python ≥ 3.10.

---

## Server requirements

hivemind-core must have `hivemind-audio-binary-protocol` installed and configured. On its own, hivemind-core does not handle audio.

See [Audio Binary Protocol](../server/audio-binary-protocol.md).

---

## Quickstart

> **Pre-flight:** this satellite sends audio to hivemind-core, so **hivemind-core MUST have the [Audio Binary Protocol](../server/audio-binary-protocol.md) configured** — it provides the VAD-gated STT, TTS, and wakeword. Without it the satellite connects but its audio is silently dropped, with no error.

**1. On hivemind-core** — register a client:

```bash
hivemind-core add-client --name my-mic-sat
# note the access_key and password printed
```

**2. On the satellite** — write the identity file:

```bash
hivemind-client set-identity \
  --key <access_key> \
  --password <password> \
  --host <server_host_or_ip>
```

**3. Run:**

```bash
hivemind-mic-sat
```

Or pass credentials directly without storing them:

```bash
hivemind-mic-sat --key <key> --password <password> --host <host> --port 5678 --siteid kitchen
```

The full flag set is `--key --password --host --port --siteid`. Each falls back to the
identity file written by `hivemind-client set-identity`.

!!! note
    mic-satellite has **no** `--selfsigned` flag. If hivemind-core uses a self-signed
    certificate, prefer a plain `ws://` LAN connection or a properly issued
    certificate.

---

## Configuration

mic-satellite reads the standard OVOS configuration at `~/.config/mycroft/mycroft.conf`.

| Plugin type | Config key | Notes |
|---|---|---|
| Microphone | `listener.microphone.module` | Required |
| VAD | `listener.VAD.module` | Required |
| PHAL | `PHAL.ovos-phal-...` | Optional, platform support |

Example `mycroft.conf` excerpt:

```json
{
  "listener": {
    "microphone": {
      "module": "ovos-microphone-plugin-alsa"
    },
    "VAD": {
      "module": "ovos-vad-plugin-silero"
    }
  }
}
```

---

## How audio flows

It helps to trace one spoken sentence end to end. The device does only the first two
steps; the moment there's sound, everything else happens on the server and comes back as
audio to play:

```
[Microphone] → [VAD] → raw audio chunks → [hivemind-core]
                                              ↓
                                    [hivemind-audio-binary-protocol]
                                              ↓
                                    WakeWord → STT → IntentService
                                              ↓
                                    TTS synthesis → audio back to satellite
                                              ↓
                                    [Satellite plays TTS audio]
```

Audio is sent as `BINARY` messages with payload type `RAW_AUDIO` (type 1). hivemind-core's `AudioBinaryProtocol.handle_microphone_input()` processes the stream.

---

## Next

Set up the server side: [Audio Binary Protocol](../server/audio-binary-protocol.md).

---

## Source

Validated against the HiveMind source:

- [`hivemind_mic_sat/__init__.py`](https://github.com/JarbasHiveMind/hivemind-mic-satellite/blob/HEAD/hivemind_mic_sat/__init__.py) — CLI flags (`--key --password --host --port --siteid`) and the binary `RAW_AUDIO` transport
- [`docs/architecture.md`](https://github.com/JarbasHiveMind/hivemind-mic-satellite/blob/HEAD/docs/architecture.md) — audio flow and server-side processing
