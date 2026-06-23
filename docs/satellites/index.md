# Choosing a Satellite

HiveMind satellites form a spectrum based on **where audio processing happens**. Choose the right satellite for your hardware and use case.

## Comparison

**Most people start with the voice satellite** (everything runs locally, works with any hub) or the **mic satellite** (cheapest device, but the hub must do STT/TTS). Pick from the table below, or read the decision guide.

| Satellite | Mic | VAD | Wakeword | STT | TTS | What crosses the wire |
|---|:---:|:---:|:---:|:---:|:---:|---|
| [HiveMind-cli](cli.md) | — | — | — | — | — | Text in / text out |
| [Microcontrollers (ESP32)](microcontrollers.md) | local | local[^esp] | local[^esp] | **hub** | **hub** | Raw audio stream |
| [hivemind-mic-satellite](mic-satellite.md) | local | local | **hub** | **hub** | **hub** | Raw audio stream |
| [HiveMind-voice-relay](voice-relay.md) | local | local | local | **hub** | **hub** | Audio after wakeword |
| [HiveMind-voice-sat](voice-sat.md) | local | local | local | local | local | Text utterances only |
| [WebSpeech Browser](webspeech.md) | browser | browser | — | **hub** | **hub** | Audio from browser |

[^esp]: On-device VAD and wakeword apply to the ESP32-S3 Voice PE build (ESP-SR WakeNet9); the plain ESP32 and the MicroPython client capture the mic and let the hub do the rest. See [Microcontrollers](microcontrollers.md).

The **microcontroller** clients (ESP32 in C, or MicroPython) are the thinnest *hardware* tier — below the mic satellite. They turn a tiny chip into a satellite without running OVOS or Python-on-a-PC on the device. See [Microcontrollers (ESP32)](microcontrollers.md).

## Hub requirements by satellite

| Satellite | Hub must provide |
|---|---|
| HiveMind-cli | Any hub (OVOS skills or Persona) |
| Microcontrollers (ESP32) | `hivemind-audio-binary-protocol` for the binary/base64 audio modes |
| hivemind-mic-satellite | `hivemind-audio-binary-protocol` for STT/TTS/wakeword |
| HiveMind-voice-relay | `hivemind-audio-binary-protocol` for STT/TTS |
| HiveMind-voice-sat | Any hub (sends text utterances) |
| WebSpeech Browser | `ovos-dinkum-listener >= 0.0.3a19` + `hivemind-core allow-msg "recognizer_loop:b64_audio"` |

See [Audio Binary Protocol](../server/audio-binary-protocol.md) for hub-side setup.

!!! note "Advanced: how the audio actually travels"
    The audio satellites do not all use the same transport, which is why their hub
    requirements differ:

    - **mic-satellite** and the **ESP32 binary mode** send **binary `RAW_AUDIO`**
      frames — handled by `hivemind-audio-binary-protocol` on the hub.
    - **voice-relay** sends **base64 audio over the bus**
      (`recognizer_loop:b64_transcribe`; TTS comes back as `speak:b64_audio`) — also
      provided by `hivemind-audio-binary-protocol`.
    - **WebSpeech** sends **base64 audio over the bus** as `recognizer_loop:b64_audio`,
      which is decoded directly by `ovos-dinkum-listener >= 0.0.3a19` once the hub
      allows that message — it does **not** use the binary `RAW_AUDIO` protocol.

## Decision guide

1. **Do you need voice input?**
   - No → use **HiveMind-cli**
   - Yes → continue

2. **Is the device a bare microcontroller (ESP32, Raspberry Pi Pico W)?**
   - Yes → use the [Microcontrollers (ESP32)](microcontrollers.md) clients — C/ESP-IDF
     or MicroPython
   - No → continue

3. **Can your device run local STT and TTS models?** (needs a capable CPU/GPU)
   - Yes → use **HiveMind-voice-sat** (most private, works offline after setup)
   - No → continue

4. **Do you need a wakeword on the device?** (saves bandwidth; needed for service-at-scale)
   - Yes → use **HiveMind-voice-relay** (local wakeword; STT/TTS on hub)
   - No → use **hivemind-mic-satellite** (cheapest hardware; everything on hub)

5. **Is this a web browser or web app?**
   - Yes → use **WebSpeech Browser**

## About hub-owned STT/TTS

When a satellite uses hub-side audio processing (mic-satellite or voice-relay), the hub operator decides the STT engine, TTS engine, and voice — the satellite cannot override them. This is the "HiveMind as a service" model: speech services are authenticated and centrally governed, like any other message on the protocol.

A voice-sat by contrast runs its own STT and TTS plugins and sends only the transcribed text to the hub. It has full control over its local audio stack.

## See also

- **Multi-user text chatroom** — [hivemind-flask-chatroom](https://github.com/JarbasHiveMind/hivemind-flask-chatroom)
  is a text-only Flask web app (default port `8985`, no audio) where one server-side
  credential is fanned out to many browser visitors. Handy for a shared text terminal
  to a hub; it is not a per-device satellite.

## Next

Head to the satellite that fits your device: [Microcontrollers (ESP32)](microcontrollers.md), [HiveMind-voice-sat](voice-sat.md), [hivemind-mic-satellite](mic-satellite.md), [HiveMind-voice-relay](voice-relay.md), [HiveMind-cli](cli.md), or [WebSpeech Browser](webspeech.md).

## Source

Validated against the HiveMind source:

- [`docs/configuration.md`](https://github.com/JarbasHiveMind/hivemind-webspeech/blob/HEAD/docs/configuration.md) — WebSpeech hub requirement (`ovos-dinkum-listener` + `allow-msg "recognizer_loop:b64_audio"`)
- [`hivemind_voice_relay/service.py`](https://github.com/JarbasHiveMind/HiveMind-voice-relay/blob/HEAD/hivemind_voice_relay/service.py) — voice-relay base64-over-bus transport (`recognizer_loop:b64_transcribe` / `speak:b64_audio`)
- [`hivemind_mic_sat/__init__.py`](https://github.com/JarbasHiveMind/hivemind-mic-satellite/blob/HEAD/hivemind_mic_sat/__init__.py) — mic-satellite binary `RAW_AUDIO` transport
- [`README.md`](https://github.com/JarbasHiveMind/hivemind-esp32-client/blob/HEAD/README.md) — ESP32 satellite tier
- [`README.md`](https://github.com/JarbasHiveMind/hivemind-micropython-client/blob/HEAD/README.md) — MicroPython satellite tier
- [`README.md`](https://github.com/JarbasHiveMind/hivemind-flask-chatroom/blob/HEAD/README.md) — text-only chatroom web app
