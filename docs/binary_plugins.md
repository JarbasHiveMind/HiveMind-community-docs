# Binary Plugins for HiveMind

Agent Plugins handle text-based messages. HiveMind also supports Binary Plugins for transmitting and processing binary payloads such as audio streams.

These plugins provide secure services like Speech-to-Text (STT) and Text-to-Speech (TTS) across HiveMind without exposing them directly. Binary data always travels over the same encrypted HiveMind transport as other messages.

---

## What is a Binary Plugin?

A Binary Plugin defines how HiveMind nodes exchange and process non-text payloads. It lets nodes forward microphone input, receive synthesized audio, or otherwise handle binary data within the hive.

Unlike agent plugins, which process BUS messages, binary plugins work directly with binary streams.

---

## Current implementation

The reference plugin is [hivemind-audio-binary-protocol](https://github.com/JarbasHiveMind/hivemind-audio-binary-protocol), built on OpenVoiceOS plugins.

The binary plugin system currently focuses on STT and TTS, but the design allows more use cases in the future.

- The first binary plugin implementation loads existing OpenVoiceOS TTS and STT plugins.
- This makes HiveMind compatible with hundreds of plugins in the OVOS ecosystem, without special adaptations.
- It also supports microphone plugins:
  - Audio streams go to the HiveMind server when Voice Activity Detection (VAD) triggers.
  - In this mode, wake word detection also runs centrally on the HiveMind server.

This design keeps HiveMind satellites lightweight and delegates heavy tasks to a hub or a stronger node.

---

## Example use cases

### Speech-to-Text (STT)

1. The satellite streams microphone audio as a binary payload.
2. A HiveMind server running an STT plugin transcribes the audio.
3. The transcription returns as a standard text BUS message to the satellite.

### Text-to-Speech (TTS)

1. The node sends a text string to a HiveMind TTS-capable server.
2. The TTS service generates speech audio.
3. The audio returns as a binary payload, ready for playback on the requesting node.

### Microphone plugins

- HiveMind nodes can act as remote microphones.
- Audio streams upstream only when VAD detects speech.
- Wake word detection stays centralized on the HiveMind server.

---

## Why binary plugins matter

- **Secure transport**: all binary data flows over HiveMind's encrypted transport.
- **Ecosystem compatibility**: directly supports existing OVOS STT/TTS plugins.
- **Lightweight satellites**: offload heavy STT/TTS work to a hub.
- **Flexibility**: supports microphone streaming, wake word offloading, and future binary tasks such as images or embeddings.

In short: Agent Plugins handle text input and output. Binary Plugins securely handle binary payloads, such as audio streams, over HiveMind, and enable distributed STT/TTS and microphone offloading.

---
[← Agents](agents.md) · [Home](index.md) · [Network →](network_plugins.md)
