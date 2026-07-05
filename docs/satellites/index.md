# Choosing a Satellite

A satellite is whatever you talk *to* — but not every satellite carries the same load.
At one end sits a terminal where you type and the server does absolutely everything. At
the other sits a Raspberry Pi that hears the wake word, transcribes your speech, and
speaks the reply all on its own, handing the server nothing but finished text. Every
device in between is a question of one thing: **how much of the listening happens on the
device, and how much on the server.** The thinner the satellite, the cheaper the
hardware and the more it leans on the network. The thicker it is, the more it keeps to
itself — even the audio stays home. This page helps you find your spot on that line.

!!! abstract "In a nutshell"
    - Satellites differ by **which stages run on the device** (mic, VAD, wakeword, STT, TTS) and **what actually crosses the wire** (typed text, raw audio, or audio only after the wake word fires).
    - Anything that ships audio needs a server ready to catch it: raw-audio and base64-audio clients require `hivemind-audio-binary-protocol` on hivemind-core; text-only satellites work with any server.
    - The [voice satellite](voice-sat.md) keeps all speech on the device and sends only text; the [mic satellite](mic-satellite.md) is the cheapest device to build but hands STT/TTS to the server.
    - Not sure? The [decision guide](#decision-guide) maps a device to its satellite in a few questions.

---

## Comparison

**Most people start with the voice satellite** (everything runs locally, works with any hivemind-core instance) or the **mic satellite** (cheapest device, but hivemind-core must do STT/TTS). Pick from the table below, or read the decision guide.

| Satellite | Mic | VAD | Wakeword | STT | TTS | What crosses the wire |
|---|:---:|:---:|:---:|:---:|:---:|---|
| [HiveMind-cli](cli.md) | — | — | — | — | — | Text in / text out |
| [Microcontrollers (ESP32)](microcontrollers.md) | local | local[^esp] | local[^esp] | **server** | **server** | Raw audio stream |
| [hivemind-mic-satellite](mic-satellite.md) | local | local | **server** | **server** | **server** | Raw audio stream |
| [HiveMind-voice-relay](voice-relay.md) | local | local | local | **server** | **server** | Audio after wakeword |
| [HiveMind-voice-sat](voice-sat.md) | local | local | local | local | local | Text utterances only |
| [WebSpeech Browser](webspeech.md) | browser | browser | — | **server** | **server** | Audio from browser |

[^esp]: On-device VAD and wakeword apply to the ESP32-S3 Voice PE build (ESP-SR WakeNet9); the plain ESP32 and the MicroPython client capture the mic and let hivemind-core do the rest. See [Microcontrollers](microcontrollers.md).

The **microcontroller** clients (ESP32 in C, or MicroPython) are the thinnest *hardware* tier — below the mic satellite. They turn a tiny chip into a satellite without running OVOS or Python-on-a-PC on the device. See [Microcontrollers (ESP32)](microcontrollers.md).

---

## Server requirements by satellite

| Satellite | hivemind-core must provide |
|---|---|
| HiveMind-cli | Any hivemind-core instance (OVOS skills or Persona) |
| Microcontrollers (ESP32) | `hivemind-audio-binary-protocol` for the binary/base64 audio modes |
| hivemind-mic-satellite | `hivemind-audio-binary-protocol` for STT/TTS/wakeword |
| HiveMind-voice-relay | `hivemind-audio-binary-protocol` for STT/TTS |
| HiveMind-voice-sat | Any hivemind-core instance (sends text utterances) |
| WebSpeech Browser | `ovos-dinkum-listener >= 0.0.3a19` + `hivemind-core allow-msg "recognizer_loop:b64_audio"` |

See [Audio Binary Protocol](../server/audio-binary-protocol.md) for server-side setup.

---

!!! note "Advanced: how the audio actually travels"
    The audio satellites do not all use the same transport, which is why their server
    requirements differ:

    - **mic-satellite** and the **ESP32 binary mode** send **binary `RAW_AUDIO`**
      frames — handled by `hivemind-audio-binary-protocol` on hivemind-core.
    - **voice-relay** sends **base64 audio over the bus**
      (`recognizer_loop:b64_transcribe`; TTS comes back as `speak:b64_audio`) — also
      provided by `hivemind-audio-binary-protocol`.
    - **WebSpeech** sends **base64 audio over the bus** as `recognizer_loop:b64_audio`,
      which is decoded directly by `ovos-dinkum-listener >= 0.0.3a19` once hivemind-core
      allows that message — it does **not** use the binary `RAW_AUDIO` protocol.

---

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
   - Yes → use **HiveMind-voice-relay** (local wakeword; STT/TTS on hivemind-core)
   - No → use **hivemind-mic-satellite** (cheapest hardware; everything on hivemind-core)

5. **Is this a web browser or web app?**
   - Yes → use **WebSpeech Browser**

---

## About server-owned STT/TTS

When a satellite uses server-side audio processing (mic-satellite or voice-relay), the hivemind-core operator decides the STT engine, TTS engine, and voice — the satellite cannot override them. This is the "HiveMind as a service" model: speech services are authenticated and centrally governed, like any other message on the protocol.

A voice-sat by contrast runs its own STT and TTS plugins and sends only the transcribed text to hivemind-core. It has full control over its local audio stack.

---

## See also

- **Multi-user text chatroom** — [hivemind-flask-chatroom](https://github.com/JarbasHiveMind/hivemind-flask-chatroom)
  is a text-only Flask web app (default port `8985`, no audio) where one server-side
  credential is fanned out to many browser visitors. Handy for a shared text terminal
  to a hivemind-core instance; it is not a per-device satellite.

---

## Next

Head to the satellite that fits your device: [Microcontrollers (ESP32)](microcontrollers.md), [HiveMind-voice-sat](voice-sat.md), [hivemind-mic-satellite](mic-satellite.md), [HiveMind-voice-relay](voice-relay.md), [HiveMind-cli](cli.md), or [WebSpeech Browser](webspeech.md).

---

## Source

Validated against the HiveMind source:

- [`docs/configuration.md`](https://github.com/JarbasHiveMind/hivemind-webspeech/blob/HEAD/docs/configuration.md) — WebSpeech server requirement (`ovos-dinkum-listener` + `allow-msg "recognizer_loop:b64_audio"`)
- [`hivemind_voice_relay/service.py`](https://github.com/JarbasHiveMind/HiveMind-voice-relay/blob/HEAD/hivemind_voice_relay/service.py) — voice-relay base64-over-bus transport (`recognizer_loop:b64_transcribe` / `speak:b64_audio`)
- [`hivemind_mic_sat/__init__.py`](https://github.com/JarbasHiveMind/hivemind-mic-satellite/blob/HEAD/hivemind_mic_sat/__init__.py) — mic-satellite binary `RAW_AUDIO` transport
- [`README.md`](https://github.com/JarbasHiveMind/hivemind-esp32-client/blob/HEAD/README.md) — ESP32 satellite tier
- [`README.md`](https://github.com/JarbasHiveMind/hivemind-micropython-client/blob/HEAD/README.md) — MicroPython satellite tier
- [`README.md`](https://github.com/JarbasHiveMind/hivemind-flask-chatroom/blob/HEAD/README.md) — text-only chatroom web app
