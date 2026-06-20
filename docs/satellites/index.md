# Choosing a Satellite

HiveMind satellites form a spectrum based on **where audio processing happens**. Choose the right satellite for your hardware and use case.

## Comparison

**Most people start with the voice satellite** (everything runs locally, works with any hub) or the **mic satellite** (cheapest device, but the hub must do STT/TTS). Pick from the table below, or read the decision guide.

| Satellite | Mic | VAD | Wakeword | STT | TTS | What crosses the wire |
|---|:---:|:---:|:---:|:---:|:---:|---|
| [HiveMind-cli](cli.md) | — | — | — | — | — | Text in / text out |
| [hivemind-mic-satellite](mic-satellite.md) | local | local | **hub** | **hub** | **hub** | Raw audio stream |
| [HiveMind-voice-relay](voice-relay.md) | local | local | local | **hub** | **hub** | Audio after wakeword |
| [HiveMind-voice-sat](voice-sat.md) | local | local | local | local | local | Text utterances only |
| [WebSpeech Browser](webspeech.md) | browser | browser | — | **hub** | **hub** | Audio from browser |

## Hub requirements by satellite

| Satellite | Hub must provide |
|---|---|
| HiveMind-cli | Any hub (OVOS skills or Persona) |
| hivemind-mic-satellite | `hivemind-audio-binary-protocol` for STT/TTS/wakeword |
| HiveMind-voice-relay | `hivemind-audio-binary-protocol` for STT/TTS |
| HiveMind-voice-sat | Any hub (sends text utterances) |
| WebSpeech Browser | `hivemind-audio-binary-protocol` for STT/TTS |

See [Audio Binary Protocol](../server/audio-binary-protocol.md) for hub-side setup.

## Decision guide

1. **Do you need voice input?**
   - No → use **HiveMind-cli**
   - Yes → continue

2. **Can your device run local STT and TTS models?** (needs a capable CPU/GPU)
   - Yes → use **HiveMind-voice-sat** (most private, works offline after setup)
   - No → continue

3. **Do you need a wakeword on the device?** (saves bandwidth; needed for service-at-scale)
   - Yes → use **HiveMind-voice-relay** (local wakeword; STT/TTS on hub)
   - No → use **hivemind-mic-satellite** (cheapest hardware; everything on hub)

4. **Is this a web browser or web app?**
   - Yes → use **WebSpeech Browser**

## About hub-owned STT/TTS

When a satellite uses hub-side audio processing (mic-satellite or voice-relay), the hub operator decides the STT engine, TTS engine, and voice — the satellite cannot override them. This is the "HiveMind as a service" model: speech services are authenticated and centrally governed, like any other message on the protocol.

A voice-sat by contrast runs its own STT and TTS plugins and sends only the transcribed text to the hub. It has full control over its local audio stack.

## Next

Head to the satellite that fits your device: [HiveMind-voice-sat](voice-sat.md), [hivemind-mic-satellite](mic-satellite.md), [HiveMind-voice-relay](voice-relay.md), [HiveMind-cli](cli.md), or [WebSpeech Browser](webspeech.md).
