# Microphone Satellite

The thinnest HiveMind satellite. Runs microphone capture and Voice Activity Detection (VAD) on-device; everything else — wakeword detection, STT, intent handling, TTS synthesis — runs on the hub.

**What runs locally**: Microphone + VAD

**What the hub provides**: Wakeword, STT, intent processing, TTS (requires `hivemind-audio-binary-protocol` on the hub)

## When to use it

- Devices with very limited CPU/RAM (Raspberry Pi Zero, recycled phones)
- Scenarios where you want zero local model downloads
- Personal homelab with a handful of devices on a fast LAN

## When not to use it

mic-satellite streams every detected voice segment to the hub continuously (VAD-gated but not wakeword-gated). This is bandwidth-intensive and puts the full STT load on the server for all speech activity. It is appropriate for a small homelab. For a deployment serving multiple users, prefer [voice-relay](voice-relay.md) — local wakeword means audio only leaves the device after activation.

## Install

```bash
pip install hivemind-mic-satellite
```

Requires Python ≥ 3.10.

## Hub requirements

The hub must have `hivemind-audio-binary-protocol` installed and configured. Plain `hivemind-core` does not handle audio.

See [Audio Binary Protocol](../server/audio-binary-protocol.md).

## Quickstart

> **Pre-flight:** this satellite sends audio to the hub, so **the hub MUST have the [Audio Binary Protocol](../server/audio-binary-protocol.md) configured** — it provides the VAD-gated STT, TTS, and wakeword. Without it the satellite connects but its audio is silently dropped, with no error.

**1. On the hub** — register a client:

```bash
hivemind-core add-client --name my-mic-sat
# note the access_key and password printed
```

**2. On the satellite** — write the identity file:

```bash
hivemind-client set-identity \
  --key <access_key> \
  --password <password> \
  --host <hub_host_or_ip>
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
    mic-satellite has **no** `--selfsigned` flag. If your hub uses a self-signed
    certificate, prefer a plain `ws://` LAN connection or a properly issued
    certificate.

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

## How audio flows

```
[Microphone] → [VAD] → raw audio chunks → [HiveMind Hub]
                                              ↓
                                    [hivemind-audio-binary-protocol]
                                              ↓
                                    WakeWord → STT → IntentService
                                              ↓
                                    TTS synthesis → audio back to satellite
                                              ↓
                                    [Satellite plays TTS audio]
```

Audio is sent as `BINARY` messages with payload type `RAW_AUDIO` (type 1). The hub's `AudioBinaryProtocol.handle_microphone_input()` processes the stream.

## Next

Set up the hub side: [Audio Binary Protocol](../server/audio-binary-protocol.md).

## Source

Validated against the HiveMind source:

- [`hivemind_mic_sat/__init__.py`](https://github.com/JarbasHiveMind/hivemind-mic-satellite/blob/HEAD/hivemind_mic_sat/__init__.py) — CLI flags (`--key --password --host --port --siteid`) and the binary `RAW_AUDIO` transport
- [`docs/architecture.md`](https://github.com/JarbasHiveMind/hivemind-mic-satellite/blob/HEAD/docs/architecture.md) — audio flow and hub-side processing
